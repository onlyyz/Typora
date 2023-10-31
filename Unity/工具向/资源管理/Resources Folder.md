<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=z1wPBbK2g9o

#### 1.1加载资源

```c#
//Load a text file (Assets/Resources/Text/textFile01.txt)
var textFile = Resources.Load<TextAsset>("Text/textFile01");

//Load text from a JSON file (Assets/Resources/Text/jsonFile01.json)
var jsonTextFile = Resources.Load<TextAsset>("Text/jsonFile01");
//Then use JsonUtility.FromJson<T>() to deserialize jsonTextFile into an object

//Load a Texture (Assets/Resources/Textures/texture01.png)
var texture = Resources.Load<Texture2D>("Textures/texture01");

//Load a Sprite (Assets/Resources/Sprites/sprite01.png)
var sprite = Resources.Load<Sprite>("Sprites/sprite01");

//Load an AudioClip (Assets/Resources/Audio/audioClip01.mp3)
var audioClip = Resources.Load<AudioClip>("Audio/audioClip01");
```

