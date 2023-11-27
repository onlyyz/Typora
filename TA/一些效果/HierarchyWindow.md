```C#
//ColorPalette

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace CustomHierarchyUI
{
    [System.Serializable]
    public class HierarchyFontColorDesign
    {
        // [Tooltip("Rename gameObject begin with this keychar")]
        [Tooltip("关键词识别")]
        public string keyChar;
        // [Tooltip("Don't forget to change alpha to 255")]
        [Tooltip("字体颜色")]
        public Color textColor;
        // [Tooltip("Don't forget to change alpha to 255")]
        [Tooltip("背景颜色")]
        public Color backgroundColor;
        [Tooltip("字体位置")]
        public TextAnchor textAlignment;
        [Tooltip("字体粗细")]
        public FontStyle fontStyle;
    }
    
    [CreateAssetMenu(menuName = "Custom/UI")]
    public class ColorPalette : ScriptableObject
    {
        public List<HierarchyFontColorDesign> colorDesigns = new List<HierarchyFontColorDesign>();
    }
}
```



```C#
using UnityEditor;
using UnityEngine;

namespace CustomHierarchyUI
{
    [InitializeOnLoad]
    public class StyleHierarchy
    {
        //Find ColorPalette GUID
        static string[] dataArray;
        //Get ColorPalette(ScriptableObject) path
        static string path;
        static ColorPalette colorPalette;

        static StyleHierarchy()
        {
            //根据脚本获取
            dataArray = AssetDatabase.FindAssets("t:ColorPalette");

            if (dataArray.Length >= 1)
            {   
                //我们只有一个调色板，因此使用 dataArray[0] 获取文件路径
                //We have only one color palette, so we use dataArray[0] to get the path of the file
                path = AssetDatabase.GUIDToAssetPath(dataArray[0]);
                // Debug.Log(path);
                colorPalette = AssetDatabase.LoadAssetAtPath<ColorPalette>(path);
                EditorApplication.hierarchyWindowItemOnGUI += OnHierarchyWindow;
            }
        }
        private static void OnHierarchyWindow(int instanceID, Rect selectionRect)
        {
            //To make sure there is no error on the first time the tool imported in project
            if (dataArray.Length == 0) return;
            // Debug.Log(dataArray.Length);
            // Debug.Log(colorPalette);
         
            Object instance = EditorUtility.InstanceIDToObject(instanceID);
            // Debug.Log(instance.name);
            if (instance != null)
            {
                // Debug.Log(colorPalette.colorDesigns.Count);
                for (int i = 0; i < colorPalette.colorDesigns.Count; i++)
                {
                    var design = colorPalette.colorDesigns[i];

                    //Check if the name of each gameObject is begin with keyChar in colorDesigns list.
                    if (instance.name.StartsWith(design.keyChar))
                    {
                        //Remove the symbol(keyChar) from the name.
                        string newName = instance.name.Substring(design.keyChar.Length);
                        //Draw a rectangle as a background, and set the color.
                        EditorGUI.DrawRect(selectionRect, design.backgroundColor);

                        //Create a new GUIStyle to match the desing in colorDesigns list.
                        GUIStyle newStyle = new GUIStyle
                        {
                            alignment = design.textAlignment,
                            fontStyle = design.fontStyle,
                            normal = new GUIStyleState()
                            {
                                textColor = design.textColor,
                            }
                        };
                        //Draw a label to show the name in upper letters and newStyle.
                        //If you don't like all capital latter, you can remove ".ToUpper()".
                        EditorGUI.LabelField(selectionRect, newName.ToUpper(), newStyle);
                    }
                }
            }
        }
    }
}
```

