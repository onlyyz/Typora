想问一问关于UE开发custom shader/rendering的问题，在不能改UE引擎的情况下(例如需要上架UE market place)，我要把一个unity URP的toon shader插件移值UE，有没有什么地方是Unity能轻易做到，但在UE很难/几乎做不到的?我想知道的有这一些点:

1 custom material inspector (= 类似unity的MaterialPropertyDrawer)2) 最传统

2 pass方法的outline (= unity的shader加outline Pass{})

3 shader_feature,multi_compile (或类似的功能)

4 画一些类似GBuffer的prepass (= URP的renderer feature和pass)

5 每个角色独立的shadow map (= URP的renderer feature和pass)

6 改shader能很快看到改动(editor中compile很快)

7 自动生成tangent space smoothed normal，储存在mesh未使用的UV(float2)，给outline用的 (= unity的AssetPostprocessor->OnPostprocessModel)，不能在project window生出新的asset我UE经验实在不足，所以想提前理解什么是能做的，什么是100%不可能做的。谢谢各位