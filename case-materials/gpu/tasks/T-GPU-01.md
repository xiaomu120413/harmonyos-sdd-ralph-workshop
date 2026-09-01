# T-GPU-01｜真机选择 OHOS Hardware Decoder

- Requirement：FreeRDP H.264 subsystem 在目标设备选择 `OHOS-AVCodec` 硬件 decoder。
- Allowed：build flags、subsystem registration、capability/selection logs。
- Forbidden：修改合成、Surface owner、AVC444。
- RED/Probe：当前运行日志无法证明 decoder 或显示 software fallback。
- Verify：decoder name、capability、configure/start 与 fallback reason。
- Stop：设备不提供硬件 H.264 能力则返回 `UNKNOWN/UNSUPPORTED`。

