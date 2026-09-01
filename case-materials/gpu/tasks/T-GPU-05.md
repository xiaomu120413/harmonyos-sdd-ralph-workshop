# T-GPU-05｜生命周期与恢复

- Source Scope：decoder/surface target generation、pause/detach/recreate、stale worker task rejection、owner release/reclaim；不重建整个业务 session 来掩盖局部状态错误。
- Requirement：resize、切后台、Surface 重建和重连后拒绝旧 buffer，并恢复可见输出。
- Allowed：decoder/Surface lifecycle、generation、release/recreate。
- Forbidden：为了恢复重建整个业务 session，除非 RFC 明确要求。
- Verify：surfaceGeneration、targetGeneration、stale task rejection、fallback/recovery。
- Stop：旧 target 仍可 present 时判 FAIL。
- Exit Gate：resize、前后台、Surface 重建、重连至少各有一条 generation/owner/fallback trace；恢复可见但旧任务仍可写入不得判 PASS。
