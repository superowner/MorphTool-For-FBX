# MorphTool-For-FBX

first, you need https://github.com/x3bits/libmmd-for-unity
then you mast rewrite switch(_ModelType){} to fit your model
and last drop PlayMorph.cs and set LoadVmd()

-------------------------------------------------------------------------------------------------------------
----------PlayMorph.cs---------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------

using LibMMD.Motion;
using LibMMD.Reader;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayMorph : MonoBehaviour
{
    public enum ModelType
    {
        MMD,Blender,Maya
    }
    public ModelType _ModelType= ModelType.MMD;
    public static PlayMorph Instance;
    public Transform Actor;
    public float DelayTime=0f;//second
    public AudioSource _AudioSource;
    [HideInInspector]
    public float Max = 100;
    [HideInInspector]
    public float Min = 0;

    private bool IsStartAudio;
    private float Counter = 0;
    private SkinnedMeshRenderer[] SkinnedMeshRenderers;
    private Dictionary<string, Vector2Int> MorphDic = new Dictionary<string, Vector2Int>();
    private MmdMotion _motion;
    private double _playTime;
    private bool IsPlay = false;
    private Dictionary<string, string> MorphMap = new Dictionary<string, string>();
    private HashSet<string> MorphNames = new HashSet<string>();

    void Awake()
    {
        Instance = this;
    }
    
    public void LoadVmd(string filePath)
    {
        _motion = new VmdReader().Read(filePath);
        if (_motion.Length != 0)
        {
            switch(_ModelType)
            {
                case ModelType.MMD://mmd4ma
                    {
                        foreach (var key in MorphDic.Keys)
                        {
                            int index= key.IndexOf('.')+1;//不同模型表情切割规则
                            //print(key.Substring(index));
                            try
                            {
                                MorphMap[key.Substring(index)] = key;
                            }
                            catch { }
                        }
                        foreach (var name in MorphMap.Keys)
                        {
                            if (_motion.IsMorphRegistered(name))
                            {
                                MorphNames.Add(name);
                            }
                        }
                    }
                    break;
            }
            _playTime = 0;
            IsPlay = true;
        }
        else IsPlay = false;
    }

    void Update()
    {
        if (IsPlay)
        {
            if(Counter< DelayTime)
                Counter += Time.deltaTime;
            else
            {
                if(!IsStartAudio&&this._AudioSource)
                {
                    this._AudioSource.Play();
                    IsStartAudio = true;
                }
            }

            _playTime += Time.deltaTime;
            foreach (var name in MorphNames)
            {
                var morphPose = _motion.GetMorphPose(name, _playTime);
                Play(MorphMap[name], morphPose.Weight);
            }
        }
    }

    // Start is called before the first frame update
    void Start()
    {
        SkinnedMeshRenderers = Actor.GetComponentsInChildren<SkinnedMeshRenderer>();
        for (int i = 0; i < SkinnedMeshRenderers.Length; ++i)
        {
            int index = SkinnedMeshRenderers[i].sharedMesh.blendShapeCount;
            if(index>0)
            {
                for (int m = 0; m < index; ++m)
                {
                    MorphDic[SkinnedMeshRenderers[i].sharedMesh.GetBlendShapeName(m)] = new Vector2Int(i,m);
                }
            }
        }
        if (null != this._AudioSource)
            this._AudioSource.playOnAwake = false;

        //test
        LoadVmd(Application.dataPath+"/lamb.vmd");
    }

    public void Play(string name,float weight)
    {
        //print("name:"+ name+ " weight:" + weight);
        weight = Mathf.Clamp(weight, Min, Max);
        if(MorphDic.TryGetValue(name, out Vector2Int m_index))
        {
            SkinnedMeshRenderers[m_index.x].SetBlendShapeWeight(m_index.y, weight*100);
        }
    }
}


