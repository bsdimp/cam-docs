Note: This needs to be split into SIM and Periph docs.

# CAM Error Recovery

From time to time a SIM may need to undertake an error recovery process during which I/O to the SIM's hardware (or part of it) isn't possible.

When the SIM decides it's time to start a recovery process, the general model in use in CAM is to freeque the queues so that no further I/Os happen until the recovery process is over. However, there's a number of races that need to be considered to robustly implement this correctly.

At some point, the SIM declares that it's entering error recovery. Some, or all, of the periphs will not be able to schedule I/O operations while the error recovery is in process.

Three things must happen
 1. Freeze the simq
 1. Set internal state so that all new I/Os that arrive have to be turned back to the submitter. This is to ensure that any CCBs that were allocated before the error recovery was declared can be robustly dealth with when they get submitted to *sim_action()*.
 1. Walk the list of commands as part of the recovery process, canceling them in the device (if applicable), cancel their I/O timeout, freeze the devq for that periph and return them for requeue.

To freeze the queue, do the following for each CCB that you need to either turn away, or return after cancelling.
```
    ccb->ccb_h.status = CAM_REQUEUE_REQ | CAM_DEV_QFRZN;
    xpt_freeze_devq(ccb->ccb_h.path, 1 /*count*/);
    xpt_done(ccb);
```

Now the recovery will happen. It will clear out the CCBs that are in the SIM, and not allow any new CCBs into the SIM until the error recovery is done. A correleary of this action is that if the SIM's error recovery decides that it can't carry on and has to remove all the periphs, no CCBs will be in flight. The SIM is responsible for clearing out any periph drivers by calling *cam_periph_invalidate* on them.

Once the error recovery is complete, you need to clear the internal state, unfreeze the simq by calling *xpt_release_simq*.

![uncached image](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.github.com/bsdimp/cam-docs/main/cam_sim_error_recovery.uml)

