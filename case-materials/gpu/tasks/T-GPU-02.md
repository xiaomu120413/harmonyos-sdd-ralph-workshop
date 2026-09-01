# T-GPU-02｜AVC420 一帧合法输出

- Requirement：一个已知 AVC420 sample 产生可验证 output。
- Allowed：OHOS codec buffer adapter、format/stride/plane diagnostics。
- Forbidden：AVC444、zero-copy、队列重构。
- Verify：frameId、input bytes、output size/format/stride/planes、bounded wait。
- PASS：output 合法且能关联输入；callback 成功但内容不明为 `UNKNOWN`。
- Stop：输出格式不在支持集合时保留原始描述并 fallback。

