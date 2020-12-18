# Unity に関するメモ

## スクリプト関連

***

### プライベート変数をインスペクタに表示する

変数の前に *[SerializeField]* を付けることで、 public でない変数もインスペクタに表示することができる

```C#
[SerializeField]
float value;
```

逆に、 public だが、インスペクタに表示したくない場合は、 *[HideInInspector]* を変数の前に付ける。

```C#
[HideInInspector]
float value;
```

***

### 主なコールバック関数

**Start**  
スクリプトがアクティブになった最初のフレームで呼び出されるメソッド。

**Update**  
カメラのレンダリング前に毎フレーム呼び出されるメソッド。

**Awake**  
Start が呼び出される前に呼び出されるメソッド。( GetComponent や、 FindObjectOfType などを使って参照をセットアップするのに最適なタイミング)

**LateUpdate**  
すべてのスクリプトの Update が呼び出された後に呼び出されることが保証されているメソッド。

**OnDestroy**  
オブジェクトが破棄されるときに呼び出されるメソッド。

**OnBecomeVisible**  
Renderer コンポーネントがアタッチされているオブジェクトが、カメラの視界に入ったときに呼び出されるメソッド。逆に *OnBecomeInvisible* は、視界から消えたときに呼び出されるメソッド。

※オブジェクトごとのメソッドの呼び出し順序は保証されていませんが、 Edit > Settings > Script Execution Order で指定することができる。

***

### フレームレートに依存しない処理を行う

*Time.deltaTime* に前回のフレーム更新からの経過時間がセットされるので、それを利用して処理を行うことでフレームレートに依存しない移動処理などが可能になる。  
※*Time.timeScale* を使用すると、ゲーム全体を速くしたり遅くしたりできる。

***

### ゲームオブジェクトにアタッチされているコンポーネントを取得する

**GetComponent<コンポーネント名>()** を使用すると、アタッチされているコンポーネントを取得できる。取得できない場合は null を返すので注意。  
例

```C#
    var renderer = GetComponent<Renderer>();
```

**GetComponents<コンポーネント名>()** で同一のコンポーネントの配列を取得できる。

**GetComponentInChildren<コンポーネント名>()** で子コンポーネントを含めた最初のコンポーネントを取得する。

**GetComponentsInChildren<コンポーネント名>()** は上記の配列版。

**GetComponentInParent<コンポーネント名>()** は親コンポーネントを含めた最初のコンポーネントを取得する。

**CetComponentsInParent<コンポーネント名>()** は上記の配列版。

***

### オブジェクトの検索

**FindObjectOfType<オブジェクト型>()** を使ってシーン内のオブジェクトを見つけることができる。
**FindObjectsOfType<オブジェクト型>()** で配列で取得。  
※見つからない場合は null を返すことに注意。

***

### 複数フレームにまたがる処理

コルーチンを使用することで複数フレームにまたがる処理ができる。コルーチンは、 IEnumerater を返す任意のメソッドで、 StartCoroutine で開始されます。
コルーチン内では、待ち時間を示すオブジェクトを返して一定時間処理を待ちます。

例  
コルーチン側

```C#
IEnumerator LogAfterDelay() {
    Debug.Log("Back in a second!");
    yield return new WaitForSeconds(1); // 1秒後に処理を再開する
    Debug.Log("I'm back!");
}
```

呼び出し側

```C#
    StartCoroutine(LogAfterDelay());    // コルーチンを呼び出す
```

戻り値を IEnumerator にすることで、 Start メソッドもコルーチンにすることが可能。

```C#
IEnumerator Start() {
    ...
}
```

コルーチンにはパラメータを渡すこともできる。  
コルーチン側

```C#
IEnumerator Func(int value) {
    ...
}
```

呼び出し側

```C#
    StartCoroutine(Func(10));
```

コルーチンで null を返すと、1フレーム待つ事ができます。

```C#
IEnumerator Func() {
    yield return null;  // null を返すことで1フレーム待つ
    ...
}
```

yield break を使用してコルーチンを終了することができる。

```C#
IEnumerator Func() {
    while (true) {
        ...
        if (...) {
            yield break;    // コルーチンを終了する
        }
    }
}
```

WaitForSecond の他にもコルーチンで使用できる関数があります。

WaitForEndOfFrame は、すべてのカメラのレンダリングが終わって、画面が更新される前まで待機します。  
これは、スクリーンショットを作成する場合などに使用できます。

WaitForSecondsRealtime は、WaitForSeconds 同様に一定の秒数待ちますが、 Time.timeScale の影響を受けないリアルタイム時間を使用します。

WailUntil と WaitWhile は、メソッドを呼び出した結果によって実行を継続するかどうかを決定します。

WaitWhile は、指定関数が true を返すまで停止します。  
例

```C#
    yield return new WaitWhile(() => transform.position.y < 5); // y が 5 未満になるまで待機する
```

WaitUntil は、指定関数が false を返すまで停止します。  
例

```C#
    yield return new WaitUntil(() => transform.position.y < 5); // y が 5 以上になるまで待機する
```

***

### シングルトンを使う

スクリプトをシングルトンにするには、 public で自身のインスタンスを保持します。その変数に Awake でインスタンスを設定します。

例  
シングルトンクラス定義

```C#
public class SimpleSingleton : MonoBehaviour {
    public static SimpleSingleton instance; // static で自身のインスタンスを保持する

    void Awake() {
        instance = this;    // Awake でインスタンスに自身を設定する
    }

    public void DoSomething() {
        Debug.Log("シングルトンの関数");
    }
}
```

シングルトンクラスの使用例

```C#
    SimpleSingleton.instance.DoSomething(); // シングルトンクラスのメソッド呼び出し
```

これで、このゲームオブジェクトが存在する限り DoSomething はどこからでもアクセス可能になります。

***

### シーンをロードする

シーンをロードするには、シーンを登録する必要があります。 File > Build Settings を開いて、 Scene In Build にプロジェクトからシーンをドラッグアンドドロップして登録します。

シーンをロードするには、スクリプトに下記の import を追加します。

```C#
using UnityEngine.SceneManagement;
```

スクリプト内で、 SceneManager.LoadScene を使用してシーンを読み込みます。

```C#
    SceneManager.LoadScene("SceneName");
```

この方法でシーンをロードすると、ロード完了までゲームは停止しますが、 LoadSceneAsync メソッドを使用すると、バックグラウンドでシーンを詠み込むことができます。

例

```C#
public void LoadLevelAsync() {
    // バックグラウンドでシーンロードを開始し、読み込み状態を示すオブジェクトを受け取る
    var operation = SceneManager.LoadSceneAsync("Game");
    Debug.Log("Starting load...");

    // ロード完了時に自動的にシーン表示を行わない
    operation.allowSceneActivation = false;

    // シーン読み込み完了まで他の処理をするためにコルーチンを呼び出す
    StartCoroutine(WaitForLoading(operation));
}
    
IEnumerator WaitForLoading(AsyncOperation operation) {
    // シーンを 90% 詠み込むまで待つ
    while (operation.progress < 0.9f) {
        yield return null;
    }

    // 読み込み完了
    Debug.Log("Loading complete!");
    // シーン読み込み完了でシーン表示を実行するように設定
    operation.allowSceneActivation = true;
}
```

シーンは複数同時に読み込めます。そのためには、 LoadScene を呼び出すときに、 LoadSceneMode を Additive に設定します。これにより、シーンがロードされ、現在ロードされているシーンに追加されます。

```C#
    public void LoadLevelAdditive() {
        // 現在のシーンに追加でシーンを詠み込む
        SceneManager.LoadScene("Game", LoadSceneMode.Additive);
    }
```

追加で読み込まれたシーンを削除するには、 UnloadSceneAsync を使います。

```C#
public void UnloadLevel() {
    // バックグラウンドでシーンをアンロードする
    var unloadOperation = SceneManager.UnloadSceneAsync("Game");

    // アンロード中になにか処理をするためにコルーチンを呼び出す
    StartCoroutine(WaitForUnloading(unloadOperation));
}

IEnumerator WaitForUnloading(AsyncOperation operation) {
    // シーンの削除完了まで待つ
    yield return new WaitUntil(() => operation.isDone);
    // シーンが使用していたアセットも削除する場合は下記のようにする
    Resources.UnloadUnusedAssets();
}
```

エディタ上でも、シーンの上で右クリックして Open Scene Additive を選択すると、複数のシーンを詠み込むことができます。

***

### ファイル保存のパスを取得する

Application.persistentDataPath プロパティーを使用してデータ保存パスを取得できます。

```C#
public string PathForFilename(string filename) {
    // Application.persistentDataPath を使用してデータの保存パスを取得する
    var folderToStoreFilesIn = Application.persistentDataPath;

    // System.IO.Path.Combine を使用してパスにファイル名を追加する
    var path = System.IO.Path.Combine(folderToStoreFilesIn, filename);

    return path;
}
```

***

### ゲームの状態の保存と読み込み

ここでは、データを JSON 形式で保存します。JSON を使用するために、 LitJSON というオープンソースライブラリを使用します。
LitJSON は、 <https://github.com/LitJSON/litjson/releases/latest> から ZIP ファイルをダウンロードして、 src フォルダの内容をアセット内にコピーして使用します。

まず、新しい C# スクリプトを作成し、内容を削除し先頭に下記のコードを追加します。

```C#
using LitJson;                      // LitJSON を使用して JSON の読み書きを行います
using System.IO;                    // System.IO を使用してファイルの入出力を行います
using System.Linq;                  // System.Linq を使用してシーン内のすべての保存可能なスクリプトの検索処理を簡素化します
using UnityEngine.SceneManagement;  // ゲームのロードは新しいシーンのロードを意味するため UnityEngine.SceneManagement クラスで処理します
```

LINQ を使用するとメモリが動的に割り当てられるので、意図しないタイミングでガベージコレクションが実行される可能性があるため、ゲーム中は使わないほうが良いのですが、データのセーブ・ロード時にはゲームは停止していることが多いため、使用することにします。  
処理が終わったあとには、 System.GC.Collect を実行して、ゲーム中のガベージコレクション実行を回避します。

次に、スクリプトに次のコードを追加します。

```C#
// ISavable インターフェース定義
public interface ISaveable {
    // ユニークなID
    string SaveID { get; }

    // セーブデータ取得用プロパティー
    JsonData SavedData { get; }

    // データ読み込み用メソッド
    void LoadFromData(JsonData data);
}

public static class SavingService {
    // 定数定義
    private const string ACTIVE_SCENE_KEY = "activeScene";
    private const string SCENES_KEY = "scenes";
    private const string OBJECTS_KEY = "objects";
    private const string SAVEID_KEY = "$saveID";

    // シリアライズデータをファイル書き込み可能パスに fileName として作成する
    public static void SaveGame(string fileName) {
        // データ書き込み用 JsonData 作成
        var result = new JsonData();

        // MonoBehaviour の中で ISaveable を含むものを検索
        var allSaveableObjects = Object
            .FindObjectsOfType<MonoBehaviour>()
            .OfType<ISaveable>();

        if (allSaveableObjects.Count() > 0) {
            // オブジェクトリスト格納用の JsonData 作成
            var savedObjects = new JsonData();

            // セーブするオブジェクト数分繰り返す
            foreach (var saveableObject in allSaveableObjects) {
                // オブジェクトのセーブデータ取得
                var data = saveableObject.SavedData;

                // オブジェクトの確認
                if (data.IsObject) {
                    // セーブ ID 保存
                    data[SAVEID_KEY] = saveableObject.SaveID;

                    // 保存オブジェクトに追加
                    savedObjects.Add(data);
                } else {
                    // 保存できない警告出力
                    var behaviour = saveableObject as MonoBehaviour;

                    Debug.LogWarningFormat(
                        behaviour,
                        "{0}'s save data is not a dictionary. The " +
                        "object was not saved.",
                        behaviour.name
                        );
                }
            }

            // 保存オブジェクトのリストを書き込み用 JsonData に保存
            result[OBJECTS_KEY] = savedObjects;
        } else {
            // 保存対象がない
            Debug.LogWarningFormat(
                "The scene did not include any saveable objects.");
        }

        // 現在のシーン保存処理

        // オープンされているシーン保存用 JsonData 作成
        var openScenes = new JsonData();

        // シーン数を取得し、シーン名を追加していく
        var sceneCount = SceneManager.sceneCount;

        for (int i = 0; i < sceneCount; i++) {
            var scene = SceneManager.GetSceneAt(i);

            openScenes.Add(scene.name);
        }

        // オープンシーンリスト保存
        result[SCENES_KEY] = openScenes;

        // アクティブなシーン名保存
        result[ACTIVE_SCENE_KEY] = SceneManager.GetActiveScene().name;

        // ファイルパス作成
        var outputPath = Path.Combine(
            Application.persistentDataPath, fileName);

        // JSON 出力を読みやすい形に設定
        var writer = new JsonWriter();
        writer.PrettyPrint = true;
        
        // JSON テキストに変換
        result.ToJson(writer);

        // ディスクへの書き込み
        File.WriteAllText(outputPath, writer.ToString());
        
        // 保存場所出力
        Debug.LogFormat("Wrote saved game to {0}", outputPath);

        // 参照を開放し、ガベージコレクションを強制的にを実行する
        result = null;
        System.GC.Collect();
    }
}
```

次は、この JSON ファイルの読み込み処理です。  
Unity では SceneManager.LoadScene でシーンを読み込んだ場合、関数終了時点では完全にシーン読み込みが完了していないため、その時点でシーンに変更を加えると上書きされてしまいます。  
それを回避するために、シーンの読み込みが実際に完了したあとに実行されるコードを登録する必要があります。

```C#
    // シーン読み込み完了後に実行するデリゲートへの参照定義
    static UnityEngine.Events.UnityAction<Scene, LoadSceneMode> LoadObjectsAfterSceneLoad;

    // データのロード処理
    public static bool LoadGame(string fileName) {
        // ファイルパス設定
        var dataPath = Path.Combine(Application.persistentDataPath, fileName);

        // ファイルの存在確認
        if (File.Exists(dataPath) == false) {
            Debug.LogErrorFormat("No file exists at {0}", dataPath);
            return false;
        }

        // JSON データの読み込み
        var text = File.ReadAllText(dataPath);
        var data = JsonMapper.ToObject(text);

        // データ形式の検証
        if (data == null || data.IsObject == false) {
            Debug.LogErrorFormat(
                "Data at {0} is not a JSON object", dataPath);
            return false;
        }

        // シーン検出
        if (!data.ContainsKey("scenes")) {
            Debug.LogWarningFormat(
                "Data at {0} does not contain any scenes; not " +
                "loading any!",
                dataPath
            );
            return false;
        }

        // シーンリスト取得
        var scenes = data[SCENES_KEY];

        int sceneCount = scenes.Count;

        if (sceneCount == 0) {
            Debug.LogWarningFormat(
                "Data at {0} doesn't specify any scenes to load.",
                dataPath
            );
            return false;
        }

        // 各シーンの読み込み
        for (int i = 0; i < sceneCount; i++) {
            var scene = (string)scenes[i];

            // 最初のシーンならすべてのシーンを置き換える
            if (i == 0) {
                SceneManager.LoadScene(scene, LoadSceneMode.Single);
            } else {
                // そうでなければシーンの追加
                SceneManager.LoadScene(scene, LoadSceneMode.Additive);
            }
        }

        // アクティブなシーンを設定する
        if (data.ContainsKey(ACTIVE_SCENE_KEY)) {
            var activeSceneName = (string)data[ACTIVE_SCENE_KEY];
            var activeScene = SceneManager.GetSceneByName(activeSceneName);
            if (activeScene.IsValid() == false) {
                Debug.LogErrorFormat(
                    "Data at {0} specifies an active scene that " +
                    "doesn't exist. Stopping loading here.",
                    dataPath
                );
                return false;
            }

            SceneManager.SetActiveScene(activeScene);
        } else {
            // 最初のシーンをアクティブにする警告
            Debug.LogWarningFormat("Data at {0} does not specify an " +
                "active scene.", dataPath);
        }

        // シーン内の全オブジェクトのロード
        if (data.ContainsKey(OBJECTS_KEY)) {
            var objects = data[OBJECTS_KEY];

            // シーン読み込み完了時に実行する処理のデリゲート定義

            LoadObjectsAfterSceneLoad = (scene, loadSceneMode) => {
                // シーン ID とオブジェクトのディクショナリ作成
                var allLoadableObjects = Object
                    .FindObjectsOfType<MonoBehaviour>()
                    .OfType<ISaveable>()
                    .ToDictionary(o => o.SaveID, o => o);

                // 詠み込むオブジェクトのリスト作成
                var objectsCount = objects.Count;

                // 各アイテム処理
                for (int i = 0; i < objectsCount; i++) {
                    // セーブデータ取得
                    var objectData = objects[i];

                    // セーブ ID 取得
                    var saveID = (string)objectData[SAVEID_KEY];

                    // セーブ ID が一致するオブジェクト検索
                    if (allLoadableObjects.ContainsKey(saveID)) {
                        var loadableObject = allLoadableObjects[saveID];

                        // オブジェクトデータの読み込み
                        loadableObject.LoadFromData(objectData);
                    }
                }

                // 自身の呼び出し削除
                SceneManager.sceneLoaded -= LoadObjectsAfterSceneLoad;

                // デリゲート参照開放
                LoadObjectsAfterSceneLoad = null;

                // ガベージコレクション強制実行
                System.GC.Collect();
            };

            // シーン読み込み後実行処理(上記)の登録
            SceneManager.sceneLoaded += LoadObjectsAfterSceneLoad;
        }
        return true;
    }
```

ゲームを保存するには、 SavingService の SaveGame メソッドを呼び出します。

```C#
    // "SaveGame.json" というファイルにゲームを保存します
    SavingService.SaveGame("SaveGame.json");
```

ゲームの読み込みには、 LoadGame メソッドを呼び出します。

```C#
    // "SaveGame.json" というセーブデータを読み込みます
    SavingService.LoadGame("SaveGame.json");
```

次に、保存できるオブジェクトの抽象クラスを作成します。

```C#
    public abstract class SaveableBehavior : MonoBehaviour,
        ISaveable,                      // セーブ可能なクラスにする
        ISerializationCallbackReceiver  // シーンファイルがエディターに保存時に実行する処理を実装する
    {
        // SaveData と LoadFromData はサブクラスで実装するのでこのクラスには実装しません
        public abstract JsonData SavedData { get; }
        public abstract void LoadFromData(JsonData data);

        // Unity はシーンファイル保存時に自動プロパティを保存しないため、下記のようにします。
        public string SaveID {
            get {
                return _saveID;
            }
            set {
                _saveID = value;
            }
        }

        // Unity エディターが保存するように [SerializeField] 属性をつけます。
        // また、 Unity エディターのインスペクターに表示されないように [HideInInspector] 属性もつけます。
        [HideInInspector]
        [SerializeField]
        private string _saveID;

        // OnBeforeSerialize は Unity がこのオブジェクトをシーンファイルとして保存する前に呼び出されます。
        public void OnBeforeSerialize() {
            // ID 設定済みチェック
            if (_saveID == null) {
                // 新規に GUID を設定する
                _saveID = System.Guid.NewGuid().ToString();
            }
        }

        // OnAfterDeserialize は、 Unity がこのオブジェクトをシーンファイルとして読み込んだあとに呼び出されます。
        public void OnAfterDeserialize() {
            // ここでは何も処理しませんが、 ISerializationCallbackReceiver を実装するにはこのメソッドが存在していなければなりません。
        }
    }
```

保存可能なオブジェクトの例は以下のようになります。

```C#
    public class TransformSaver : SaveableBehaviour {
        // 定数定義
        private const string LOCAL_POSITION_KEY = "localPosition";
        private const string LOCAL_ROTATION_KEY = "localRotation";
        private const string LOCAL_SCALE_KEY = "localScale";

        // SerializeValue は、 Unity がシリアルライズ方法を知っているオブジェクトを
        // セーブデータに含める JsonData に変換するヘルパー関数です
        private JsonData SerializeValue(object obj) {
            // これは非効率ですが、 Unity のオブジェクトのシリアライス処理を省けます
            return JsonMapper.ToObject(JsonUtility.ToJson(obj));
        }

        // DeserializeValue は Unity の既存の型を JsonData からオブジェクトに変換します
        private T DeserializeValue<T>(JsonData data) {
            return JsonUtility.FromJson<T>(data.ToJson());
        }
        
        // このコンポーネントのセーブデータを提供する
        public override JsonData SavedData {
            get {
                // セーブ用 JsonData オブジェクトを作成する
                var result = new JsonData();

                // position 、 rotation 、 scale を保存
                result[LOCAL_POSITION_KEY] =
                    SerializeValue(transform.localPosition);

                result[LOCAL_ROTATION_KEY] = SerializeValue(transform.localRotation);
                result[LOCAL_SCALE_KEY] =
                    SerializeValue(transform.localScale);

                return result;
            }
        }

        // ロードデータがある場合はコンポーネントの状態に反映する
        public override void LoadFromData(JsonData data) {
            // データが完全な形で保存されている保証はありません。ちゃんとチェックしましょう

            // データに各アイテムが含まれているかチェックする

            // position 更新
            if (data.ContainsKey(LOCAL_POSITION_KEY)) {
                transform.localPosition =
                    DeserializeValue<Vector3>(data[LOCAL_POSITION_KEY]);
            }

            // rotation 更新
            if (data.ContainsKey(LOCAL_ROTATION_KEY)) {
                transform.localRotation =
                    DeserializeValue<Quaternion>(data[LOCAL_ROTATION_KEY]);
            }

            // scale 更新
            if (data.ContainsKey(LOCAL_SCALE_KEY)) {
                transform.localScale =
                    DeserializeValue<Vector3>(data[LOCAL_SCALE_KEY]);
            }
        }
    }
```

***

### オブジェクトプールを使ったオブジェクト管理

オブジェクトが不要になったときに非アクティブ化し、必要になったときに再アクティブ化するオブジェクトプールを作成します。

```C#
    // インターフェース定義
    public interface IObjectPoolNotifier {
        // オブジェクトがプールに返されるときに呼び出される
        void OnEnqueuedToPool();

        // オブジェクトがプールから取り出されるか、作成直後の場合に呼び出される
        // created が true の場合、オブジェクトは作成されたばかりであることを示す
        void OnCreatedOrDequeuedFromPool(bool created);
    }

    // オブジェクトプール実装
    public class ObjectPool : MonoBehaviour {
        // インスタンス化されるプレハブ
        [SerializeField]
        private GameObject prefab;

        // 未使用オブジェクトのキュー
        private Queue<GameObject> inactiveObjects = new Queue<GameObject>();

        // プールからオブジェクトを取得する
        // 1つもない場合は新規に作成される
        public GameObject GetObject() {
            // 再利用アイテム存在チェック
            if (inactiveObjects.Count > 0) {
                // キューからオブジェクトを取得
                var dequeuedObject = inactiveObjects.Dequeue();

                // 親オブジェクトをプールからルートに変更する
                dequeuedObject.transform.parent = null;
                dequeuedObject.SetActive(true);

                // IObjectPoolNotifier を実装した MonoBehaviour に
                // プールからのオブジェクト取得を通知する

                var notifiers = dequeuedObject
                    .GetComponents<IObjectPoolNotifier>();

                foreach (var notifier in notifiers) {
                    // プールからの取得をスクリプトに通知する
                    notifier.OnCreatedOrDequeuedFromPool(false);
                }

                // 使用するオブジェクトを返す
                return dequeuedObject;
            } else {
                // プールにオブジェクトがないので新規に作成する

                var newObject = Instantiate(prefab);

                // プールに戻せるようにプールタグを追加する
                var poolTag = newObject.AddComponent<PooledObject>();

                poolTag.owner = this;

                // プールタグがインスペクターに表示されないように設定する
                poolTag.hideFlags = HideFlags.HideInInspector;
                
                // 新規作成をオブジェクトに通知する
                var notifiers = newObject
                    .GetComponents<IObjectPoolNotifier>();

                foreach (var notifier in notifiers) {
                    // 新規作成されたことをスクリプトに通知する
                    notifier.OnCreatedOrDequeuedFromPool(true);
                }

                // 作成したオブジェクトを返す
                return newObject;
            }
        }
        
        // オブジェクトを無効にしてキューに入れる
        public void ReturnObject(GameObject gameObject) {
            // 通知が必要なオブジェクト検索
            var notifiers = gameObject
                .GetComponents<IObjectPoolNotifier>();

            foreach (var notifier in notifiers) {
                // プールに戻ったことを通知する
                notifier.OnEnqueuedToPool();
            }
            
            // オブジェクトを無効化し、子にする
            gameObject.SetActive(false);
            gameObject.transform.parent = this.transform;
            
            // オブジェクトをキューに追加する
            inactiveObjects.Enqueue(gameObject);
        }
    }

    // ReturnToPool メソッドによって使用される
    public class PooledObject : MonoBehaviour {
        public ObjectPool owner;
    }

// GameObject クラスに ReturnToPool メソッドを追加するクラス
public static class PooledGameObjectExtensions {
    // オブジェクトを作成元のプールに戻す
    public static void ReturnToPool(this GameObject gameObject) {
        // PooledObject を探す
        var pooledObject = gameObject.GetComponent<PooledObject>();

        // 存在チェック
        if (pooledObject == null) {
            // 存在しない場合はプールから作成されていない
            Debug.LogErrorFormat(gameObject,
                "Cannot return {0} to object pool, because it was not"   +
                "created from one.",
                gameObject);
            return
        }

        // オブジェクト返却をプールに通知
        pooledObject.owner.ReturnObject(gameObject);
    }
}
```

テストコードです。

```C#
// オブジェクトプールの使用例
public class ObjectPoolDemo : MonoBehaviour {
    // オブジェクト取得用オブジェクトプール
    [SerializeField]
    private ObjectPool pool;

    IEnumerator Start() {

        // 0.1 〜 0.5 秒ごとにプールからオブジェクトを取得して配置する
        while (true) {

            // プールからオブジェクトを取得(または新規作成)
            var o = pool.GetObject();

            // 半径 4 の球のどこかの点を取得
            var position = Random.insideUnitSphere * 4;

            // 配置
            o.transform.position = position;

            // 0.1 ～ 0.5 秒後に繰り返す
            var delay = Random.Range(0.1f, 0.5f);

            yield return new WaitForSeconds(delay);
        }
    }
}

// 1 秒待機してプールに戻るオブジェクトの例
public class ReturnAfterDelay : MonoBehaviour, IObjectPoolNotifier {
    // プールからの削除または新規作成時ハンドラ
    public void OnCreatedOrDequeuedFromPool(bool created) {
        Debug.Log("Dequeued from object pool!");

        StartCoroutine(DoReturnAfterDelay());
    }

    // プールに戻ったときに呼び出される
    public void OnEnqueuedToPool() {
        Debug.Log("Enqueued to object pool!");
    }
    
    IEnumerator DoReturnAfterDelay() {
        // 1 秒待ってプールに戻る
        yield return new WaitForSeconds(1.0f);

        // プールに戻す
        gameObject.ReturnToPool();
    }
}
```

***

### ScriptableObject を使用したアセット内へのデータ保存

```C#
// Asset > Create にエントリを作成する
[CreateAssetMenu]

// 親クラスを MonoBehaviour から ScriptableObject に変更することを忘れないでください！
public class ObjectColor : ScriptableObject {
    public Color color;
}

// 使用例
public class SetColorOnStart : MonoBehaviour {
    // データを取得する ScriptableObject
    [SerializeField]
    ObjectColor objectColor;

    private void Update() {
        // objectColor が存在しない場合は使用しない
        if (objectColor == null) {
            return;
        }

        GetComponent<Renderer>().material.color = objectColor.color;
    }
}

```

***

## 入力

***

### キーボード入力処理

Input クラスの GetKeyDown 、 GetKeyUp 、 GetKey メソッドを使用して、押されているキーを確認します。

```C#
if (Input.GetKeyDown(KeyCode.A)) {
    Debug.Log("The A key was pressed!");
}

if (Input.GetKey(KeyCode.A)) {
    Debug.Log("The A key is being held down!");
}

if (Input.GetKeyUp(KeyCode.A)) {
    Debug.Log("The A key was released!");
}

if (Input.anyKeyDown) {
    Debug.Log("A key was pressed!");
}
```

***

### マウス入力処理

Input クラスの GetMouseButtonUp 、 GetMouseButtonDown 、 GetMouseButton メソッドを使用して、キーボードのキーにアクセスする方法と同様のボタン状態にアクセスします。

```C#
if (Input.GetMouseButtonDown(0)) {
    Debug.Log("Left mouse button was pressed!");
}

if (Input.GetMouseButton(0)) {
    Debug.Log("Left mouse button is being held down!");
}

if (Input.GetMouseButtonUp(0)) {
    Debug.Log("Left mouse button was released!");
}
```

マウスボタンに加えて、 Input クラスの GetAxis メソッドを使用してマウスの動きにアクセスすることもできます。

```C#
var mouseX = Input.GetAxis("Mouse X");

var mouseY = Input.GetAxis("Mouse Y");

Debug.LogFormat("Mouse movement: {0},{1}", mouseX, mouseY);
```

GetAxis は、 -1 から 1 の範囲の数値を返します。

mousePosition プロパティを使用して、画面上のマウスの位置にアクセスすることもできます。

このプロパティは、画面の解像度に依存する画面座標で報告されます。 画面の解像度に依存しない位置を使用する場合は、メインのカメラオブジェクトに位置をビューポート座標に変換するように依頼します。

```C#
var mousePosition = Input.mousePosition;

var screenSpacePosition =
    Camera.main.ScreenToViewportPoint(mousePosition);
```

***

### マウスカーソルのロックと非表示

*Cursor.lockState* プロパティを *CursorLockMode.Locked* または *CursorLockMode.Confined* に設定してマウスをゲーム画面に固定し、 *Cursor.visible* を false に設定して、カーソルを非表示にします。

```C#
    // 画面またはウィンドウの中央にカーソルをロックします
    Cursor.lockState = CursorLockMode.Locked;

    // カーソルがウィンドウから離れないようにします
    Cursor.lockState = CursorLockMode.Confined;

    // マウスカーソルをロックしない
    Cursor.lockState = CursorLockMode.None;

    // マウスカーソルを非表示にする
    Cursor.visible = false;
```

※エディターでは、 Esc キーでカーソルを再度有効にできます。

***

### ゲームパッド処理

Xbox や PlayStation のコントローラを使用したい場合の処理です。

接続されたコントローラーを検出するには、 *Input.GetJoystickName* を使用します。

```C#
    var names = Input.GetJoystickNames();

    Debug.LogFormat("Joysticks: {0}", names);
```

コントローラーを接続すると、キーボードと同じように、 *GetKey* 系のメソッドを使用してボタンの状態を取得できます。

例

```C#
    // コントローラーのボタンのリスト( 1 つ目のコントローラー)
    var buttons = new [] {
        KeyCode.Joystick1Button0,
        KeyCode.Joystick1Button1,
        KeyCode.Joystick1Button2,
        KeyCode.Joystick1Button3,
        KeyCode.Joystick1Button4,
        KeyCode.Joystick1Button5,
        KeyCode.Joystick1Button6,
        KeyCode.Joystick1Button7,
        KeyCode.Joystick1Button8,
        KeyCode.Joystick1Button9,
        KeyCode.Joystick1Button10,
        KeyCode.Joystick1Button11,
        KeyCode.Joystick1Button12,
        KeyCode.Joystick1Button13,
        KeyCode.Joystick1Button14,
        KeyCode.Joystick1Button15,
        KeyCode.Joystick1Button16,
        KeyCode.Joystick1Button17,
        KeyCode.Joystick1Button18,
        KeyCode.Joystick1Button19
    };
    
    // すべてのボタンの状態をルーブで取得
    foreach (var button in buttons) {
        if (Input.GetKeyDown(button)) {
            Debug.LogFormat("Button {0} pressed", button);
        }

        if (Input.GetKeyUp(button)) {
            Debug.LogFormat("Button {0} released", button);
        }
    }
```

GetAxis を使用してジョイスティックの位置を取得することもできます。

例

```C#
    Debug.LogFormat(
        "Primary Joystick: X: {0}; Y:{1}",
        Input.GetAxis("Horizontal"),
        Input.GetAxis("Vertical")
    );
```

***

### Unity の入力システムのカスタマイズ

Unity では、特定のキーにマップされる仮想のボタンを定義できます。

定義されたボタンを表示するには、 **Edit > Settings > Input** を選択して入力設定画面に移動します。

プロジェクトのデフォルトで定義されているボタンと軸のリストが表示されます。

例えばジャンプボタンがスペースバーに割り当てられている場合、 *Input* クラスの *GetButtonDown* 、 *GetButton* 、 *GetButtonUp* メソッドを使用してボタンの状態を取得できます。

例

```C#
    if (Input.GetButtonDown("Jump")) {
        Debug.LogFormat("Jump button was pressed!");
    }

    if (Input.GetButton("Jump")) {
        Debug.LogFormat("Jump button is being held down!");
    }
    
    if (Input.GetButtonUp("Jump")) {
        Debug.LogFormat("Jump button was released!");
    }
```

Type を Joystick Axis にして、 JoyNum を設定すると、ジョイスティックの軸を設定可能です。

***

### イベントシステムのポインタイベントへの応答

オブジェクト上でのマウス移動やボタンクリックを検知するには、イベントシステムを使用します。イベントシステムを使用するには、 GameObject メニューで EventSystem コンポーネントがアタッチされたオブジェクトを作成します。

**GameObject** メニューを開き、**UI > EventSystem** を選択すると、新しい EventSystem オブジェクトが作成されます。

Canvas を作成する場合は、同時に EventSystem も作成します。 UI も同様の方法を使用するためです。

次に、カメラ位置からカーソル位置を通り、シーンに至る線をトレースするために、レイキャスターコンポーネントを作成します。この線がなにかのコライダーに当たった場合、イベントシステムはターゲットオブジェクトにイベントをディスパッチします。

メインカメラオブジェクトを選択肢ます。

**Component** メニューを開き、 **Event > Physics Raycaster** を選択肢ます。

ここでは、 3D の物理システムを使用していますが、 2D の場合も同じ手法が使えます。 2D の場合は、 **Physics Raycaster** の代わりに、 **Physics 2D Raycaster** をカメラに使用します。 2D と 3D 両方必要な場合は、 2 つ追加することもできます。

ここでは、スクリプトがアタッチされているオブジェクト上にカーソルが移動したことを検出し、そのレンダラーの色を変更するスクリプトを作成します。

次の using を追加します。

```C#
using   UnityEngine.EventSystems ;
```

ObjectMouseInteraction クラスを下記のように置き換えます。

```C#
public class ObjectMouseInteraction :
    MonoBehaviour,
    IPointerEnterHandler,   // オブジェクトにマウスカーソルが入ったときの処理
    IPointerExitHandler,    // オブジェクトからマウスカーソルが離れたときの処理
    IPointerUpHandler,      // オブジェクト上でマウスボタンが話されたときの処理
    IPointerDownHandler,    // オブジェクト上でマウスボタンが押下されたときの処理
    IPointerClickHandler    // オブジェクト上でマウスがクリックされたときの処理
    {
        Material material;
        
        void Start() {
            material = GetComponent<Renderer>().material;
        }

        public void OnPointerClick(PointerEventData eventData) {
            Debug.LogFormat("{0} clicked!", gameObject.name);
        }
        
        public void OnPointerDown(PointerEventData eventData) {
            Debug.LogFormat("{0} pointer down!", gameObject.name);
            material.color = Color.green;
        }
        
        public void OnPointerEnter(PointerEventData eventData) {
            Debug.LogFormat("{0} pointer enter!", gameObject.name);
            material.color = Color.yellow;
        }
        
        public void OnPointerExit(PointerEventData eventData) {
            Debug.LogFormat("{0} pointer exit!", gameObject.name);
            material.color = Color.white;
        }
        
        public void OnPointerUp(PointerEventData eventData) {
            Debug.LogFormat("{0} pointer up!", gameObject.name);
            material.color = Color.yellow;
        }
    }
```

このスクリプトがアタッチされたオブジェクト上でマウスを動かしたりクリックしたりすると色が変わります。

コライダーのないオブイジェクトには反応しないことに注意が必要です。

***

## 数学

***

### ベクトルを使用したさまざまな次元の座標の保存

2 次元のベクトルは Vector2 型で定義します。

```C#
    Vector2 direction = new Vector2(0.0f, 2.0f);

    var up      = Vector2.up;       // ( 0,  1)
    var down    = Vector2.down;     // ( 0, -1)
    var left    = Vector2.left;     // (-1,  0)
    var right   = Vector2.right;    // ( 1,  0)
    var one     = Vector2.one;      // ( 1,  1)
    var zero    = Vector2.zero;     // ( 0,  0)
```

3 次元のベクトルは Vector3 型で定義します。

```C#
    Vector3 point = new Vector3(1.0f, 2f, 3.5f);
    
    var up      = Vector3.up;       // ( 0,  1,  0)
    var down    = Vector3.down;     // ( 0, -1,  0)
    var left    = Vector3.left;     // (-1,  0,  0)
    var right   = Vector3.right;    // ( 1,  0,  0)
    var forward = Vector3.forward;  // ( 0,  0,  1)
    var back    = Vector3.back;     // ( 0,  0, -1)
    var one     = Vector3.one;      // ( 1,  1,  1)
    var zero    = Vector3.zero;     // ( 0,  0,  0)
```

すべての Transform コンポーネントには、現在の回転状態に関連するローカルの方向ベクトルが定義されています。例えば、オブジェクトの前方方向は下記のようになります。

```C#
    var myForward = transform.forward;
```

ベクトルの加算は下記のようにします。

```C#
    var v1 = new Vector3(1f, 2f, 3f);
    var v2 = new Vector3(0f, 1f, 6f);

    var v3 = v1 + v2;                   // (1, 3, 9)
```

減算も同様です。

```C#
    var v4 = v2 - v1;   // (-1, -1, 3)
```

ベクトルの大きさは、 *magnitude* で取得できます。これは成分の 2 乗の合計の平方根です。

```C#
    var forwardMagnitude = Vector3.forward.magnitude;   // = 1

    var vectorMagnitude = new Vector3(2f, 5f, 3f).magnitude;    // = 6.16
```

大きさが 1 のベクトルは、単位ベクトルと呼ばれます。

*magnitude* を使用して 2 つのベクトル感の距離を計算するには、ベクトルを減算してその大きさを計算します。

```C#
    var point1 = new Vector3(5f, 1f, 0f);
    var point2 = new Vector3(7f, 0f, 2f);
    
    var distance = (point2 - point1).magnitude; // = 3
```

組み込み関数の *Distance* は同様の計算を行います。

```C#
    Vector3.Distance(point1, point2);
```

ベクトルの大きさを求めるには平方根の計算が必要ですが、 2 つの長さの比較をしたい場合は、平方根の計算を省略して大きさの 2 乗で処理できます。この方法を行うには、 *sqrMagniture* プロパティーを使用します。

```C#
    var distanceSquared = (point2 - point1).sqrMagnitude;   // = 9
```

ベクトルの自身の大きさで除算することで正規化されたベクトルが取得できます。

```C#
    var bigVector = new Vector3(4, 7, 9);   // magnitude = 12.08

    // 単位ベクトルに変換(正規化)
    var unitVector = bigVector / bigVector.magnitude;   // magnitude = 1
```

上記の値は、 *normalized* プロパティーを使用して取得可能です。

```C#
    var unitVector2 = bigVector.normalized;
```

ベクトルをスケーリングするには、ベクトルに単一の数値(スカラー)を掛けることで行われます。

```C#
    var v1 = Vector3.one * 4;   // = (4, 4, 4)
```

*Scale* メソッドを使用して、コンポーネントごとのスケーリングを行うことができます。

```C#
    v1.Scale(new Vector3(3f, 1f, 0f));  // = (12f, 4f, 0f)
```

2 つのベクトルの内積を取得することもできます。これは、指している方向の違いを示します。内積は、 2 つのベクトルの積の合計です。つまり、A \* B = sum(A.x \* B.x, A.y \* B.y, A.z \* B.z)になります。

内積を使用すると、 2 つのベクトルの類似性を判断できます。

同じ方向を示す 2 つのベクトルの内積は **1** です。

```C#
    var parallel = Vector3.Dot(Vector3.left, Vector3.left); // 1
```

反対方向を向いた 2 つのベクトルの内積は **-1** です。

```C#
    var opposite = Vector3.Dot(Vector3.left, Vector3.right);    // -1
```

互いに直角の 2 つのベクトルの内積は **0** です。

```C#
    var orthogonal = Vector3.Dot(Vector3.up, Vector3.forward);  // 0
```

2 つのベクトルの内積は、 2 つのベクトル間の角度の余弦でもあります。

これは、 2 つのベクトル間の内積のアークコサインを計算することで、ベクトル間の角度を計算できることを意味します。

```C#
    var orthoAngle = Mathf.Acos(orthogonal);
    var orthoAngleDegrees = orthoAngle * Mathf.Rad2Deg; // = 90
```

*Mathf.Acos* はラジアンの値を返すので、角度に変換するために *Mathf.Rad2Deg* 定数を掛けます。

内積を使用すると、オブジェクトが他のオブジェクトの前にあるか後ろにあるかを判断することができます。

Unity では、ローカルの Z 軸は前向きの方向を表し、オブジェクトの *Transform* の *forward* プロパティーで取得できます。

最初のオブジェクトの位置から、 2 番目のオブジェクトの位置を引く事で、最初のオブジェクトから 2 番目のオブジェクトへの方向を示すベクトルが生成できます。

次に、最初のオブジェクトの前方向との内積を取ります。

この結果と、内積に関する法則を利用して、 2 番目のオブジェクトが最初のオブジェクトの前にあるかどうかを判断できます。

同じ方向を示す 2 つのベクトルの内積が 1 であるので、 2 つ目のオブジェクトが真正面にある場合は 1 になります。

0 の場合は、オブジェクトは前方向に対して 90 度の方向にあります。

-1 の場合は、真後ろにあることになります。

```C#
    var directionToOtherObject = someOtherObjectPosition - transform.position;
    var differenceFromMyForwardDirection =
        Vector3.Dot(transform.forward, directionToOtherObject);

    if (differenceFromMyForwardDirection > 0) {
        // オブジェクトは前にある
    } else if (differenceFromMyForwardDirection < 0) {
        // オブジェクトは後ろにある
    } else {
        // 完全に直角の位置にある
    }
```

2 つの入力ベクトルに(直角に)直交する 3 番目のベクトルである外積も使用できます。

```C#
    var up = Vector3.Cross(Vector3.forward, Vector3.right);
```

外積は、 3 次元ベクトルに対してのみ使用できます。

また、 2 つのベクトル間を一定速度で移動するためのベクトルを取得することもできます。

```C#
    var moved = Vector3.MoveTowards(Vector3.zero, Vector3.one, 0.5f);
    // = (0.3, 0.3, 0.3) (大きさが 0.5 のベクトル)
```

また、法線で定義された平面からの反射を得るには次のようにします。

```C#
    var v = Vector3.Reflect(new Vector3(0.5f, -1f, 0f), Vector3.up);
    // = (0.5, 1, 0)
```

また、0 から 1 の数値を指定して、 2 つの入力ベクトル間で線形補間(lerp)することもできます。 0 を指定すると最初のベクトルを取得し、 1 を指定すると 2 番目のベクトルになり、 0.5 を指定すると 2 つのベクトルの中間になります。

```C#
    var lerped = Vector3.Lerp(Vector3.zero, Vector3.one, 0.65f);
    // = (0.65, 0.65, 0.65)
```

0 から 1 の範囲外の値を指定すると、 *lerp* は、 0 から 1 の間にクランプしますが、 *LerpUnclamped* を使用するとこれを防ぐことができます。

```C#
    var unclamped = Vector3.LerpUnclamped(Vector3.zero, Vector3.right, 2.0f);
    // = (2, 0, 0)
```

***

### 3D 空間での回転

3D 空間で回転を行う場合は、クォータニオンを使用します。

例えば、クォータニオンを使用して X 軸を中心に 90 度回転するには下記のようにします。

```C#
    var rotation = Quaternion.Euler(90, 0, 0);

    var input = new Vector3(0, 0, 1);
    var result = rotation * input;  // = (0, -1, 0)
```

回転を行わないことを示す値があります。

```C#
    var identity = Quaternion.identity;
```

*Slerp* メソッドを使用すると 2 つの角度の間を補完できます。

```C#
    var rotationX = Quaternion.Euler(90, 0, 0);

    var halfwayRotated = Quaternion.Slerp(identity, rotationX, 0.5f);
```

クォータニオンは組み合わせることができます。例えば、 Y 軸を中心に回転させていから、 X 軸を中心に回転させる場合は、それらを乗算します(逆の順で適用します)。

```C#
    var combinedRotation =
        Quaternion.Euler(90, 0, 0) *    // X 軸回転
        Quaternion.Euler(0, 90, 0);     // Y 軸回転
```

順序が重要で、 Y 軸→ X 軸にした場合は結果が異なることに注意してください。

***

### 行列を使用した 3D 空間での変換

移動、回転、スケーリングを一気に行う場合は、行列を使用します。行列は単純に数値がグリッド状になってものです。

```C#
    var matrix = new Matrix4x4();
```

グリッド内の指定位置で値の取得と設定が可能です。

```C#
    var m00 = matrix[0, 0];

    matrix[0, 1] = 2f;
```

行列にベクトルを掛けることで、移動、スケーリング、回転が可能です。

ゲームでは通常 4 × 4 の行列を使用します。

X 軸上でベクトルを 5 単位移動する行列は下記のようになります。

```C#
    var translationMatrix = new Matrix4x4(
        new Vector4(1, 0, 0, 5 ),
        new Vector4(0, 1, 0, 0),
        new Vector4(0, 0, 1, 0),
        new Vector4(1, 0, 0, 1)
    );
```

各 Vector4 は行ではなく列を示しています。

3 次元ベクトルに 4 × 4 の行列を乗算する場合、ベクトルの末尾に 1 を加えて 4 次元ベクトルを形成します。この加算成分を一般に w 成分と呼びます。

この行列に 4 次元ベクトル V を掛けると、次の結果が得られます。

```
1*Vx  +  0*Vy  +  0*Vz  +  5*Vw = resultX
0*Vx  +  1*Vy  +  0*Vz  +  0*Vw = resultY
0*Vx  +  0*Vy  +  1*Vz  +  0*Vw = resultZ
0*Vx  +  0*Vy  +  0*Vz  +  1*Vw = resultW
```

例えば、点 (0, 1, 2)(Vector3) に、この行列を書ける場合

まず、 w コンポーネントを追加します。

```text
Vx = 0, Vy = 1, Vz = 2, Vw = 1

1*0  +  0*1  +  0*2  +  5*1 = 5
0*0  +  1*1  +  0*2  +  0*1 = 1
0*0  +  0*1  +  1*2  +  0*1 = 2
0*0  +  0*1  +  0*2  +  1*1 = 1
```

次に、 4 つ目のコンポーネントを破棄して結果を取得します。つまり、最終的なベクトルは(5, 1, 2)になります。

Unuty では、この処理を Matrix4x4 型の MultiplyPoint メソッドで行います。

```C#
    var input = new Vector3(0, 1, 2);

    var result = translationMatrix.MultiplyPoint(input);    // = (5, 1, 2)
```

4 番目の要素は、透視投影のような操作のときに使います。

平行移動、回転、スケーリングなどの変換のみを行う場合は、マトリクスの一部のみを使用するため、 Matrix4x4 の MultiplyPoint4x3 を使用できます。これは少し高速ですが、平行移動、回転、スケーリングにしか使用できません。

Unity は行列を使用してポイントを変換するヘルパー関数も提供します。

```C#
    var input = new Vector3(0, 1, 2);

    var translationMatrix = Matrix4x4.Translate(new Vector3(5, 1, -2));
    var result = translationMatrix.MultiplyPoint(input);    // = (5, 2, 0)
```

行列とクォータニオンを使用して、原点を中心にポイントを回転させることもできます。

```C#
    var rotate90DegreesAroundX = Quaternion.Euler(90, 0, 0);
    
    var rotationMatrix = Matrix4x4.Rotate(rotate90DegreesAroundX);
    
    var input = new Vector3(0, 0, 1);

    var result = rotationMatrix.MultiplyPoint(input);
```

これで、ポイントは原点の前方から下に移動し、 (0, -1, 0) になります。

ベクトルが方向を表し、行列を使用してベクトルを回転させたい場合は、 *MultiplyVector* を使用できます。この方法は回転に必要な行列部分のみ使用するので若干高速になります。

```C#
    result = rotationMatrix.MultiplyVector(input);
    // = (0, -1, 0) - 同じ結果になる
```

また、原点からの距離を行列でスケーリングすることもできます。

```C#
    var scale2x2x2 = Matrix4x4.Scale(new Vector3(2f, 2f, 2f));

    var input = new Vector3(1f, 2f, 3f);

    var result = scale2x2x2.MultiplyPoint3x4(input);
    // = (2, 4, 6)
```

このように行列を組み合わせることを、行列の連結と呼びます。

```C#
    var translation = Matrix4x4.Translate(new Vector3(5, 0, 0));
    var rotation = Matrix4x4.Rotate(Quaternion.Euler(90, 0, 0));
    var scale = Matrix4x4.Scale(new Vector3(1, 5, 1));

    var combined = translation * rotation * scale;

    var input = new Vector3(1, 1, 1);
    var result = combined.MultiplyPoint(input);
    Debug.Log(result);
    // = (6, 1, 5)
```

クォータニオン同様に乗算の順序が重要です。

行列を乗算と組み合わせると、乗算の逆順で行列が適用されます。点 P 、行列 A 、B 、 C が与えられた場合下記のようになります。

```TEXT
P * (A * B * C) == (A * (B * (C * P)))
```

*Matrix4x4.TRS* メソッドを使用して、変換、回転、スケールを統合したマトリクスを作成できます。

```C#
    var transformMatrix = Matrix4x4.TRS(
        new Vector3(5, 0, 0),
        Quaternion.Euler(90, 0, 0),
        new Vector3(1, 5, 1)
    );
```

このマトリクスは、ポイントを拡大縮小、回転、平行移動します。

ローカルスペース内のコンポーネントの位置にあるポイントをワールドスペースに変換するマトリクスを取得することも可能です。

```C#
    var localToWorld = this.transform.localToWorldMatrix;
```

逆にワールドスペースからローカルスペースに変換することも可能です。

```C#
    var worldToLocal = this.transform.worldToLocalMatrix;
```

***

### アングルの操作

Transform クラスの Rotate メソッドを使用して、角度を指定して物体を回転させることができます。

```C#
    // X 軸を中心に 90 度回転する
    transform.Rotate(90, 0, 0);
```

円の角度には 360 度と 2 πラジアンという 2 つの表現方法があります。度のほうが馴染み深いですが、ラジアンのほうが計算が簡単な場合があります。

```C#
    // πラジアンの正弦(半円)は 0
    Mathf.Sin(Mathf.PI);    // = 0
```

次のように、ラジアンから度、度からラジアンに変換することができます。

```C#
    // 90 度をラジアンに変換
    var radians = 90 * Mathf.Deg2Rad;   // ~= 1.57 (π / 2)

    // 2 πラジアンを度に変換
    var degrees = 2 * Mathf.PI * Mathf.Rad2Deg; // = 360
```

2 つの単位ベクトルの内積は、それらの角度の余弦に等しくなります。度のコサインは、そのアークコサインを取得することで、元の度を取得できます。これを利用すると、次のように 2 つのベクトル間の角度を取得することができます。

```C#
    var angle = Mathf.Acos(Vector3.Dot(Vector3.up, Vector3.left));
```

この結果は、πラジアンです。ユーザーに表示する場合は度に変換する必要があります。円は 2 πラジアンと 360 度で表せますので、ラジアンから度に変換する場合は、 180/ πを掛けます。例えば、 π /2 ラジアンは、度で示すと ( π /2) \* (180/ π ) = 90 になります。
度からラジアンに変換する場合は、逆を行うのでπ /180 を掛けます。例えば、 45 度をラジアンにすると、 45 \* ( π /180) = π /4 になります。

Mathf.Deg2Rad 定数と、 Mathf.Rad2Deg 定数を使用すると、コードを単純化できます。ラジアンで示される角度に Mathf.Rad2Deg を掛けると結果は度になり、度で示される角度に Mathf.Deg2Rad を掛けると、結果はラジアンになります。

***

### ターゲットまでの距離を求める

***現在作成中***

次のようなスクリプトを作成します。

```C#
public class RangeChecker : MonoBehaviour {
    // 距離を確認する対象オブジェクト
    [SerializeField] Transform target;
    
    // この距離内にある場合は範囲内と判断する
    [SerializeField] float range = 5;
    
    // 前のフレームで範囲内かどうかを保持
    private bool targetWasInRange = false;

    void Update() {

        // オブジェクト間の距離を計算
        var distance = (target.position - transform.position).magnitude;

        if (distance <= range && targetWasInRange == false) {
            // オブジェクトが新規に範囲内に入った場合はログ出力する
            Debug.LogFormat("Target {0} entered range!", target.name);

            // 既に範囲内であるためフラグ設定
            targetWasInRange = true;
        } else if (distance > range && targetWasInRange == true) {
            // このフレームで範囲外になった場合はログ出力
            Debug.LogFormat("Target {0} exited range!", target.name);

            // 既に範囲外であるためフラグ設定
            targetWasInRange = false;
        }
    }
}
```

このスクリプトを任意のオブジェクトにアタッチして、 Target フィールドに他のオブジェクトをアタッチすると、ターゲットが範囲に出入りするタイミングを検出できます。

***

### ターゲットへの角度を求める

次のようなスクリプトを作成します。

```C#
public class AngleChecker : MonoBehaviour {

    // 角度を求めたいオブジェクト
    [SerializeField] Transform target;

    void Update() {

        // ターゲットへの正規化された方向を取得
        var directionToTarget =
            (target.position - transform.position).normalized;
            
        // 求めた方向と自身が向いている方向の内積を求める
        var dotProduct = Vector3.Dot(transform.forward,
                                     directionToTarget);

        // 角度を求める
        var angle = Mathf.Acos(dotProduct);

        // 小数点以下 1 桁までの角度をログ出力する
        Debug.LogFormat(
            "The angle between my forward direction and {0} is {1:F1}°",
            target.name, angle * Mathf.Rad2Deg
        );

    }
}
```

このオブジェクトを任意のオブジェクトにアタッチし、 Target フィールドに目的のオブジェクトを指定するとオブジェクト間の角度をログ出力します。

***

## 2D グラフィックス

### スプライトのインポート

画像をプロジェクトにドラッグ＆ドロップします。
画像を選択し、 *Texture Type* を "*Sprite (2D and UI)*" に変更します。
*Apply* をクリックすると、画像をスプライトとして使用することができます。

2D ゲーム作成時には、自動的に画像の追加はスプライトになりますが、 3D ゲーム作成時には、デフォルトではマテリアルのテクスチャとして追加されます。

***

### シーンへのスプライトの追加

画像がスプライトとして設定されている前提です。

シーンにドラッグ＆ドロップすることでシーンに追加できます。

これによってスプライトの名前のゲームオブジェクトが追加され、 *SpriteRevderer* コンポーネントを追加し、 *SpriteRenderer* を使用して表示が行われます。

*Pixels Per Unit* の値は、ピクセル単位の画像サイズとシーン内のスプライトサイズの関係を示します。スプライトが 1 ユニットあたり 1 ピクセルの場合、各ピクセルの幅と高さは 1 ユニットになります。スプライトがユニットあたり 100 ピクセルの場合は、各ピクセルの幅と高さはユニットの 1/100になります。

***

### スプライトアニメーションの作成

アニメーションとして使用する画像を選択してシーンにドラッグします。Unityは以下の処理を行います。

* 新しいアニメーターコントローラーアセットを作成し、ドラッグした画像アセットの横に保存します。
* 新しいアニメーションクリップを作成し保存場所を尋ねます。アニメーションクリップは時間経過に沿ってスプライトをループ更新するように構成されます。
* *Animator* コンポーネントと *SpriteRenderer* コンポーネントを持つ新しいゲームオブジェクトを作成します。

これでアニメーションをテストできます。

***

### 2D Physics を持つスプライトの作成

* スプライトレンダラーを持つゲームオブジェクトをシーンに追加します。
* ゲームオブジェクトを選択し、 *Add Component*をクリックします。
* *Physics 2D* > *Rigidbody 2D* を選択するか、検索フィールドに *Rigidbody2D* と入力して選択します。

これで 2D オブジェクトに質量と位置を定義して物理動作を行えます。

***

### スプライトのコリジョン形状のカスタマイズ

スプライトを選択し、 *Generate Physics Shape* がオンになっていることを確認します。

* *Sprite Editor* ボタンをクリックします。スプライトエディタウィンドウが表示されます。
* 左上のドロップダウンリストで、 *Custom Physiscs Shape* を選択します。

これで、スプライト形状の定義を開始できます。

* クリック＆ドラッグで新しいパスを追加します。
* パスのポイントをクリックして移動します。
* 2 つのポイント間をクリック＆ドラッグして、中間に新しいポイントを追加します。
* ポイントをクリックし、 Ctrl + Delete でポイントを削除します。
* 完了したら、 *Apply* をクリックします。

Unity はデフォルトでは画像の透明部分を使用して物理アウトラインを作成します。必要以上に複雑になるため、手作業で単純な形状にすることができます。

***

### 複合型コライダーの使用

* それぞれコライダーを持つゲームオブジェクトを作成します。(ボックス、ポリゴン、円などの任意のコライダー)
* 空のゲームオブジェクトを作成し、それをコライダーを持つオブジェクトの親にします。
* 親オブジェクトに、 *CompositeCollider2D* コンポーネントを追加します。
* 最後に、子ゲームオブジェクトを選択し、 *Used By Composite* チェックボックスを選択します。親オブジェクトを選択すると、その子の形状を組み合わせたコライダー形状が定義されています。

***

### Sprite Packer の使用

メモリ節約のために複数のスプライトを 1 つのテクスチャにまとめることができます。

* *Assets* メニューの *Create > Sprite Atlas* を選択して新しいスプライトアトラスを作成します。
* グループ化するスプライトを *Objects for Packing* リストにドラッグします。フォルダをドラッグすることもできます。

***

### オブジェクトへの力の適用

2D オブジェクトに物理的な力を加える方法です。

*Rigidbody* クラスの *AddForce* メソッドを使用してスクリプトで 2D オブジェクトに力を加えます。

以下のスクリプトは、プレイヤーの入力を使用してオブジェクトに力を加える方法を示します。

```C#
public class AlienForce : MonoBehaviour {

    // 加える力の強さ
    [SerializeField] float verticalForce = 1000;
    [SerializeField] float sidewaysForce = 1000;
    
    // このオブジェクトの Rigidbody2D への参照(キャッシュするため)
    Rigidbody2D body;
    
    // ゲーム開始時にリジッドボディへの参照を取得して保存する
    void Start() {
        body = GetComponent<Rigidbody2D>();
    }
    
    // FuxedUpdate で物理的な力を加えるとよりスムーズな動きになります。
    // 秒間の呼び出し回数が固定であるためです。
    void FixedUpdate() {
        // ユーザーの入力を取得し、力の強さでスケールします
        var vertical = Input.GetAxis("Vertical") * verticalForce;
        var horizontal = Input.GetAxis("Horizontal") * sidewaysForce;

        // 時間でスケールしたベクトルに変換する
        var force =
            new Vector2(horizontal, vertical) * Time.deltaTime;
            
        // スプライトに力を加えます
        body.AddForce(force);
    }
}
```

通常は *FixedUpdate* ではなく、 *Update* で処理します。*FixedUpdateは 1 フレーム間に複数回呼ばれる可能性があるためです。

スクリプトで力を加える以外にも、 *ConstantForce2D* コンポーネントを使用することもできます。このコンポーネントは、オブジェクトに継続的に力を加えます。このコンポーネントは、動きを特にコントロールする必要がないオブジェクトに適しています。

***

### コンベアベルトの作成

コンベアベルトのように、触れるオブジェクトを押しのけるオブジェクトを作成してみます。

サーフェイスエフェクタは、エフェクタのエッジに沿ってボディを押す力を適用します。

* 新たにゲームオブジェクトを作成し、*BoxCollider2D* コンポーネントを追加します。
* *User by Effector* 設定をオンにします。
* *Add Component* ボタンをクリックし、 *Physics 2D > Surface Effector 2D* コンポーネントを追加します。
* その上に、 *Rigidbody2D* と *Collider2D* が接続されているオブジェクトを配置します。
* ゲームを実行すると、オブジェクトはエフェクターに落下し、エフェクターによって押されます。

*Speed* の値が正の場合、オブジェクトは右に押され、負の場合は左に押されます。 *Speed Variation* 値を使用すると、オブジェクトに適用される速度にランダム性をもたせることができます。

***

### スプライトにカスタムマテリアルを使用する

* *Assets* メニューを開き、 *Create > Material* を選択して新しいマテリアルセットを作成します。名前を *"Sprite Diffuse"* にします。
* 新しいスプライトを選択し、インスペクターの上部で、シェーダーを *Sprites > Diffuse* に変更します。
* スプライトレンダラーがあるシーン内のゲームオブジェクトを選択し、*Sprite Diffuse* マテリアルをマテリアルスロットにドラッグします。これで照明に反応するようになります。

デフォルトのシェーダーはライトを無視します。テクスチャの色は(環境によってシェーディングされるのではなく)画面上で使用されるものです。

***

### スプライトのソート管理

スプライトの前後関係の管理についてです。

* 描画順を設定したいスプライトレンダラーを選択します。
* *Order in Layer* の値を変更します。スプライトレンダラーは、この番号の順に描画されます。小さい数字の上に大きい数字のスプライトが描画されます。

*Default* という削除できないソートレイヤーが最低 1 つは存在します。

ソートレイヤーを作成するには、 *Edit* メニューを開き、 *Project Settings > Tags & Layers* を選択します。ソートレイヤーで順序を並べ替えます。

***

### ソートグループの使用

ソートグループを使用してスプライトのソーチ順を管理します。

* 空のゲームオブジェクトを作成し、 *Sprite Group* という名前をつけます。
* それに、 *SortingGroup* コンポーネントを追加します。
* ソート状態にしたいスプライトレンダラーを移動して、このオブジェクトの子にします。

これで、同じ描画領域にある他のオブジェクトに関係なく、スプライトはこの順序になります。

***

### 2.5D シーンの作成

2D と 3D がブレンドされたシーンを作成します。

* カメラを選択し、投影(*Projection*)モードを遠近法(*Perspective*)に変更します。
* 背景オブジェクトを選択し、カメラから遠ざけます。

カメラが左右に動くと、カメラから遠い 3D オブジェクトの動きが遅くなり、奥行きが表現できます。

遠近法カメラを使用している場合、スプライトがカメラに向いていない場合ソートの問題が発生しやすいことに注意してください。

ソートグループを使用して、 2D オブジェクトが正しく並べられるようにしましょう。

***

## 3D グラフィックス

### シンプルなマテリアルの作成

* *GameObject* メニューを開き、 *3D Object > Sphere* を選択して新しい球を作成します。
* *Assets* メニューを開き、 *Create > Material* を選択し、好きな名前をつけます。
* 作成したマテリアルを選択し、色を設定します。
* マテリアルを球にドロップします。

***

### スクリプトによるマテリアルプロパティーの制御


--***現在作成中***--
