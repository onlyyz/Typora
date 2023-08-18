```C#

#region DownSample and UpSample
passName = Enum.GetName(typeof(PostFXSettings.PassEnum), settings.Pass );
cmd.BeginSample(passName);

int TwoPass = (passName == "Box" || passName == "Kawase") ? (int)settings.Pass : (int)settings.Pass+ 1;
Debug.Log(passName);
cmd.SetGlobalFloat(bloomIntensityId, 1f);
if (settings.Pass == 0)
{
    return false;
}
else{
    int i;
    for (i =0; i < bloom.maxIterations; i++)
    {
        if (desc.width == 1) 
            break;
    
        RenderingUtils.ReAllocateIfNeeded(ref m_BloomMipUp[i], desc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: m_BloomMipUp[i].name);
        RenderingUtils.ReAllocateIfNeeded(ref m_BloomMipDown[i], desc, FilterMode.Bilinear, TextureWrapMode.Clamp,name: m_BloomMipDown[i].name);
    
        //Horizontal
        Draw(templaId, m_BloomMipUp[i], (int)settings.Pass);
        //Vertical
        Draw(m_BloomMipUp[i], m_BloomMipDown[i], TwoPass);
        templaId = m_BloomMipDown[i];
    
    
        desc.width = Mathf.Max(1, desc.width >> 1);
        desc.height = Mathf.Max(1, desc.height >> 1);
    }
    
    cmd.EndSample(passName);
    
    
    cmd.SetGlobalFloat(
        bloomBucibicUpsamplingId, bloom.bicubicUpsampling ? 1f : 0f
    );
    
    templaId = m_BloomMipDown[i - 1];
    for (i -= 2; i > 0; i--) 
    {
        var mipUp = m_BloomMipDown[i];
        cmd.SetGlobalTexture(fxSource2Id, mipUp);
        Draw(templaId, mipUp, Pass.BloomCombine);
        templaId = mipUp;
    
    }
}
#endregion

```