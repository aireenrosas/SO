

/*
 * static per-instance data must live in this struct
 */
struct PS_INSTANCE_DATA {
  struct HLS_VPROC TVP;
  int PSInitState;
  SCHAR DefaultPri;
  struct HLS_SCHED_INSTANCE *Self;
  int NumThreads;
  LIST_ENTRY AllThreads;
  ULONG TimeIncrement;
};

/*
 * per-bottom-VPROC data lives in this struct
 */
struct PS_UD {
  WNT_TIME AVT, StartTime;
  ULONG Warp, Share;
  struct HLS_VPROC *udvp;
  LIST_ENTRY AllThreadsEntry;
};

/*
 * utility functions to hide null pointers from the rest of the scheduler
 */

// top virtual processor to proportional share instance data
static struct PS_INSTANCE_DATA *TVPtoPSID (struct HLS_VPROC *tvp)
{
  struct PS_INSTANCE_DATA *id;
  id = (struct PS_INSTANCE_DATA *)tvp->BottomSched->SchedData;
  return id;
}

// scheduler instance pointer to PS instance data
static struct PS_INSTANCE_DATA *SItoPSID (struct HLS_SCHED_INSTANCE *si)
{
  struct PS_INSTANCE_DATA *id;
  id = (struct PS_INSTANCE_DATA *)si->SchedData;
  return id;
}

// bottom VP to PS instance data
static struct PS_INSTANCE_DATA *BVPtoPSID (struct HLS_VPROC *bvp)
{
  struct PS_INSTANCE_DATA *id;
  id = (struct PS_INSTANCE_DATA *)bvp->TopSched->SchedData;
  return id;
}

// bottom VP to per-VP data
static struct PS_UD *BVPtoPSUD (struct HLS_VPROC *bvp)
{
  struct PS_UD *ud;
  ud = (struct PS_UD *)bvp->UpperData;
  return ud;
}

// generic message pointer to PS-specific data
static struct HLS_RR_MSG *MsgToRRMsg (MsgHandle Context)
{
  struct HLS_RR_MSG *p = (struct HLS_RR_MSG *) Context;
  return p;
}

/*
 * main body of scheduler: HLS callbacks and support functions
 */

static void RegisterPS (struct PS_INSTANCE_DATA *Inst)
{
  Inst->TVP.TopSched->CB->B_RegisterVP (Inst->TVP.TopSched, Inst->Self, &Inst->TVP);
  
  if (strncmp (Inst->TVP.TopSched->Type, "priority", 8) == 0) {
    struct HLS_RR_MSG NewMsg; 
    HLS_STATUS status;    

    NewMsg.Type = RR_MSG_SETPRI;
    NewMsg.Pri = Inst->DefaultPri;
    status = Inst->TVP.TopSched->CB->B_Msg (NULL, &Inst->TVP, (MsgHandle)&NewMsg);
  }
}

static void MakeItSo (struct PS_INSTANCE_DATA *Inst,
              WNT_TIME Now);

static HLS_STATUS PS_B_CallbackRegisterVP (struct HLS_SCHED_INSTANCE *Parent,
                       struct HLS_SCHED_INSTANCE *Child,
                       struct HLS_VPROC *bvp)
{
  struct PS_INSTANCE_DATA *Inst = BVPtoPSID (bvp);

  bvp->UpperData = (TDHandle) hls_malloc (sizeof (struct PS_UD));

  {
    struct PS_UD *ud = (struct PS_UD *) bvp->UpperData;
    ud->udvp = bvp;
    ud->Warp = 0;
    ud->AVT = 0;
    ud->Share = 0;
    InsertTailList(&Inst->AllThreads, &ud->AllThreadsEntry);
  }

  bvp->State = VP_Waiting;
  bvp->Proc = NO_PROC;

  if (Inst->NumThreads == 0) {
    RegisterPS (Inst);
    HLSSetTimer (-(10*HLS_MsToNT), Inst->Self); 
  }
  Inst->NumThreads++;

  {
    WNT_TIME Now = HLSGetCurrentTime ();
    MakeItSo (Inst, Now);
  }

  return HLS_SUCCESS;
}

static void PS_B_CallbackUnregisterVP (struct HLS_VPROC *bvp)
{
  struct PS_INSTANCE_DATA *Inst = BVPtoPSID (bvp);
  struct PS_UD *ud = BVPtoPSUD (bvp);

  Inst->NumThreads--;
  if (Inst->NumThreads == 0) {
    Inst->TVP.TopSched->CB->B_UnregisterVP (&Inst->TVP);
  }
  
  RemoveEntryList (&ud->AllThreadsEntry);

  hls_free ((void *)bvp->UpperData);
}

static void GrantIt (struct PS_INSTANCE_DATA *Inst)
{
  struct HLS_VPROC *bvp;
  struct PS_UD *ud;

  /*
   * this will have already been set to the runnable bottom VP with
   * the earliest deadline 
   */
  bvp = Inst->TVP.BottomData;

  ud = BVPtoPSUD (bvp);
  bvp->Proc = Inst->TVP.Proc;
  bvp->State = VP_Running;
  bvp->BottomSched->CB->T_VP_Grant (bvp);
  ud->StartTime = HLSGetCurrentTime ();
}

/*
 * update actual virtual time of a child VP
 */
static void UpdateAVT (struct PS_UD *ud,
               WNT_TIME Now)
{
  WNT_TIME RunTime;

  RunTime = Now - ud->StartTime;
  if (RunTime < 0) {
    RunTime = 0;
  }
  ud->AVT += 1 + (RunTime / ud->Share);
}

static void PS_T_CallbackRevoke (struct HLS_VPROC *tvp)
{
  struct PS_INSTANCE_DATA *Inst = TVPtoPSID (tvp);
  WNT_TIME Now = HLSGetCurrentTime ();
  struct HLS_VPROC *Current = tvp->BottomData;

  UpdateAVT (BVPtoPSUD (Current), Now);
  Current->State = VP_Ready;
  Current->BottomSched->CB->T_VP_Revoke (Current);
  Current->Proc = NO_PROC;
  MakeItSo (Inst, Now);
}

/*
 * return effective virtual time for a child VP
 */
static WNT_TIME EVT (struct PS_UD *t)
{
  return (t->Warp > t->AVT) ? 0 : (t->AVT - t->Warp);
}

/*
 * return pointer to runnable virtual processor with smallest effective virtual time
 */
static struct HLS_VPROC *GetVPSmallestEVT (struct PS_INSTANCE_DATA *Inst)
{
  struct PS_UD *min = NULL;
  PRLIST_ENTRY ListHead = &Inst->AllThreads;
  PRLIST_ENTRY NextEntry = ListHead->Flink;

  while (NextEntry != ListHead) {
    struct PS_UD *ud = CONTAINING_RECORD (NextEntry, struct PS_UD, AllThreadsEntry);
    NextEntry = NextEntry->Flink;
    if (ud->udvp->State != VP_Waiting &&
    (!min || EVT (ud) < EVT (min))) {
      min = ud;
    }
  }

  return (min) ? min->udvp : NULL;
}

static void PS_T_CallbackGrant (struct HLS_VPROC *tvp)
{
  struct PS_INSTANCE_DATA *Inst = TVPtoPSID (tvp);

  tvp->BottomData = GetVPSmallestEVT (Inst);
  GrantIt (Inst);
}

static void MaybePreempt (struct HLS_VPROC *Current,
              WNT_TIME Now)
{
  if (Current->State == VP_Running) {
    UpdateAVT (BVPtoPSUD (Current), Now);
    Current->State = VP_Ready;
    Current->BottomSched->CB->T_VP_Revoke (Current);
    Current->Proc = NO_PROC;
  }
}

/*
 * generic reschedule
 */
static void MakeItSo (struct PS_INSTANCE_DATA *Inst,
              WNT_TIME Now)
{
  struct HLS_VPROC *bvp = Inst->TVP.BottomData;
  struct HLS_VPROC *Next; 
  struct HLS_VPROC *Current; 

  if (bvp) {
    if (bvp->State == VP_Running) {
      struct PS_UD *ud = BVPtoPSUD (bvp);
      UpdateAVT (ud, Now);
      ud->StartTime = Now;
    }
  }

  if (Inst->TVP.State != VP_Ready) {
    Next = GetVPSmallestEVT (Inst);
  } else {
    Next = NULL;
  }
  Current = Inst->TVP.BottomData;
  
  if (Current) {
    if (Next) {
      if (Current != Next) {
    MaybePreempt (Current, Now);
    Inst->TVP.BottomData = Next;
    GrantIt (Inst);
      }
    } else {
      MaybePreempt (Current, Now);
      if (Inst->TVP.State != VP_Ready) {
    Inst->TVP.TopSched->CB->B_VP_Release (&Inst->TVP);
      }
      Inst->TVP.BottomData = NULL;
    }
  } else {
    if (Next) {
      Inst->TVP.TopSched->CB->B_VP_Request (&Inst->TVP);
    } else {
      Inst->TVP.BottomData = NULL;
      if (Inst->TVP.State == VP_Ready && !GetVPSmallestEVT(Inst)) {
    Inst->TVP.TopSched->CB->B_VP_Release (&Inst->TVP);
      }
    }
  }
}

/*
 * compute system virtual time - only used when a VP unblocks
 */
static WNT_TIME SVT (struct PS_INSTANCE_DATA *Inst)
{
  struct PS_UD *min = NULL;
  PRLIST_ENTRY ListHead = &Inst->AllThreads;
  PRLIST_ENTRY NextEntry = ListHead->Flink;

  while (NextEntry != ListHead) {
    struct PS_UD *ud = CONTAINING_RECORD (NextEntry, struct PS_UD, AllThreadsEntry);
    NextEntry = NextEntry->Flink;
    if (ud->udvp->State != VP_Waiting &&
    (!min || ud->AVT < min->AVT)) {
      min = ud;
    }
  }
  return (min) ? min->AVT : 0;
}

static void PS_B_CallbackRequest (struct HLS_VPROC *bvp)
{
  struct PS_UD *ud = BVPtoPSUD (bvp);
  struct PS_INSTANCE_DATA *Inst = BVPtoPSID (bvp);
  WNT_TIME Now = HLSGetCurrentTime ();
  WNT_TIME tSVT;

  tSVT = SVT (Inst);
  ud->AVT = max (ud->AVT, tSVT);
  bvp->State = VP_Ready;
  MakeItSo (Inst, Now);
}

static void PS_B_CallbackRelease (struct HLS_VPROC *bvp)
{
  struct PS_INSTANCE_DATA *Inst = BVPtoPSID (bvp);
  WNT_TIME Now = HLSGetCurrentTime ();

  if (bvp->State == VP_Running) {
    UpdateAVT (BVPtoPSUD(bvp), Now);
  }

  bvp->State = VP_Waiting;
  bvp->Proc = NO_PROC;
  MakeItSo (Inst, Now);
}

static void PS_I_TimerCallback (struct HLS_SCHED_INSTANCE *Self)
{
  struct PS_INSTANCE_DATA *Inst = SItoPSID (Self);
  WNT_TIME Now = HLSGetCurrentTime ();

  if (Inst->NumThreads == 0) {
    return;
  }

  MakeItSo (Inst, Now);
  HLSSetTimer (-(10*HLS_MsToNT), Inst->Self); 
}

static HLS_STATUS PS_B_CallbackMsg (struct HLS_SCHED_INSTANCE *InstArg,
                    struct HLS_VPROC *bvp,
                    MsgHandle Msg)
{
  struct PS_INSTANCE_DATA *Inst;
  struct HLS_RR_MSG *m; 
  HLS_STATUS status;

  if (bvp) {
    Inst = BVPtoPSID (bvp);
  } else {
    Inst = SItoPSID (InstArg);
  }

  m = MsgToRRMsg (Msg);

  switch (m->Type) {
  case RR_MSG_SETDEFPRI:
    Inst->DefaultPri = m->Pri;
    status = HLS_SUCCESS;
    break;
  case RR_MSG_SETPRI:
    {
      struct PS_UD *ud = BVPtoPSUD (bvp);
      
      ud->Share = m->Pri * 10;
      if (ud->Share == 0) {
    ud->Share = 10;
      }
      status = HLS_SUCCESS;
    }
    break;
  case RR_MSG_SETRR:    
    status = HLS_SUCCESS;
    break;
  case RR_MSG_SETSHARE:
    {
      struct PS_UD *ud = BVPtoPSUD (bvp);
      
      ud->Warp = m->Warp;
      ud->Share = m->Share;
      status = HLS_SUCCESS;
    }
    break;
  case RR_MSG_NULL:
    status = HLS_SUCCESS;
    break;
  default:
    status = HLS_INVALID_PARAMETER;
  }

  return status;
}

static void PS_I_Init (struct HLS_SCHED_INSTANCE *Self,
               struct HLS_SCHED_INSTANCE *Parent)
{
  Self->SchedData = (SDHandle) hls_malloc (sizeof (struct PS_INSTANCE_DATA));
  {
    struct PS_INSTANCE_DATA *Inst = (struct PS_INSTANCE_DATA *)Self->SchedData;

    InitializeListHead (&Inst->AllThreads);
    Inst->PSInitState = 1;
    Inst->NumThreads = 0;
    Inst->TimeIncrement = KeQueryTimeIncrement ();

    Inst->TVP.State = VP_Waiting;
    Inst->TVP.BottomData = NULL;
    Inst->TVP.BottomSched = Self;
    Inst->TVP.TopSched = Parent;

    strcpy (Self->Type, "priority");
    Inst->DefaultPri = -1;
    Inst->Self = Self;
  }
}

static void PS_I_Deinit (struct HLS_SCHED_INSTANCE *Self)
{
  hls_free ((void *)Self->SchedData);
}

/*
 * this structure makes callbacks available to the rest of the system
 */
struct HLS_CALLBACKS PS_CB = {
  "PS",
  PS_B_CallbackRegisterVP,
  PS_B_CallbackUnregisterVP,
  PS_B_CallbackRequest,
  PS_B_CallbackRelease,
  PS_B_CallbackMsg,
  PS_T_CallbackGrant,
  PS_T_CallbackRevoke,
  PS_I_Init,
  PS_I_Deinit,
  PS_I_TimerCallback,
};
