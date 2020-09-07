# Qt3Dエンティティの組み立て

ここでは、ゲームのビルディングブロックを、それぞれEntity要素の形で作成する作業を進めていきます。

* wall: これは、ヘビが行くことができない場所の限界を表しています。
* SnakePart: これはヘビの体の一部を表しています。
* Apple: これは、ランダムな場所でスポーンされたリンゴ（まさか！）を表しています。
* Background: これは、蛇とリンゴの後ろにあるいかした背景を表現しています。

各エンティティは、エンジンが扱うグリッド上に配置され、見つけやすいように型の識別子を持つようになります。これらのプロパティを因数分解するために、GameEntity.qmlという名前の親QMLファイルを作成してみましょう。

```QML
import Qt3D.Core 2.0

Entity {
    property int type: 0
    property vector2d gridPosition: Qt.vector2d(0, 0)
}
```

このEntity要素は、typeプロパティとgridPositionプロパティのみを定義しています。

最初に構築するのはWall.qmlファイルです。

```QML
import Qt3D.Core 2.0

GameEntity {
    id: root

    property alias position: transform.translation

    Transform {
        id: transform
    }

    components: [transform]
}
```

ご覧の通り、Wall型には視覚的な表現がありません。Raspberry Pi デバイスをターゲットにしているので、CPU/GPU の消費には十分注意しなければなりません。ゲームエリアはグリッドになっており、各セルにはエンティティのインスタンスが含まれています。蛇はWallインスタンスに囲まれています。Raspberry Piは一般的なコンピュータよりもずっと遅く、壁をすべて表示するとゲームが耐えられないほど遅くなってしまいます。

この問題に対処するために、壁は見えません。彼らは見えるビューポートの外側に配置され、ウィンドウの境界線はヘビの視覚的な限界として機能します。もちろん、Raspberry Piではなく、コンピュータをターゲットにしているのであれば、壁を表示して、何もないよりもファンシーに見えるようにしても構いません。

次に実装するEntity要素はSnakePart.qmlです。

```QML
import Qt3D.Core 2.0
import Qt3D.Render 2.0
import Qt3D.Extras 2.0

GameEntity {
    id: root

    property alias position: transform.translation

    PhongMaterial {
        id: material
        diffuse: "green"
    }

    CuboidMesh {
        id: mesh
    }

    Transform {
        id: transform
    }

    components: [material, mesh, transform]
}
```

GameAreaシーンに追加された場合、SnakePartブロックには緑色の立方体が1つ表示されます。スネークパーツブロックは、完全なヘビではなく、ヘビの体の一部です。ヘビはリンゴを食べるたびに成長することを覚えておいてください。成長とは、SnakePartのリストにSnakePartの新しいインスタンスを追加することを意味します。

Apple.qmlを進めてみましょう。

```QML
import Qt3D.Core 2.0
import Qt3D.Render 2.0
import Qt3D.Extras 2.0

GameEntity {
    id: root

    property alias position: transform.translation
    property alias color: material.diffuse

    Transform {
        id: transform
        scale: 0.5
    }

    Mesh {
        id: mesh
        source: "models/apple.obj"
    }

    DiffuseMapMaterial {
        id: material
        diffuse: "qrc:/models/apple-texture.png"
    }

    components: [material, mesh, transform]
}
```

このスニペットでは、まず Qt3D の複雑で使いやすい機能であるカスタムメッシュとそれに適用されたテクスチャを紹介します。Qt3Dでは、カスタムメッシュを読み込むためにWavefront obj形式をサポートしています。ここでは、アプリケーションの.qrcファイルに家庭料理のリンゴを追加していますが、このリソースのパスを指定するだけでロードできます。

DiffuseMapMaterial要素にも同じ原理を適用しています。カスタムテクスチャを追加し、コンポーネントのソースとして追加しました。

ご覧のように、Entityの定義とそのコンポーネントは非常によく似ています。しかし、Qt3DのCuboidMeshをカスタムモデルと簡単に交換することができました。

Background.qmlを使用して、さらに推し進めていきます。

```QML
import Qt3D.Core 2.0
import Qt3D.Render 2.0
import Qt3D.Extras 2.0

Entity {
    id: root

    property alias position: transform.translation
    property alias scale3D: transform.scale3D

    MaterialBackground {
        id: material
    }

    CuboidMesh {
        id: mesh
    }

    Transform {
        id: transform
    }

    components: [material, mesh, transform]
}
```

Backgroundブロックは、ヘビとリンゴの後ろに表示されます。一見すると、この実体はSnakePartに非常に似ています。しかし、MaterialはQt3Dのクラスではありません。シェーダーに依存するカスタム定義されたMaterialです。MaterialBackground.qmlを見てみましょう。

```QML
import Qt3D.Core 2.0
import Qt3D.Render 2.0

Material {
    id: material

    effect: Effect {
        techniques: [
            Technique {
                graphicsApiFilter {
                    api: GraphicsApiFilter.OpenGL
                    majorVersion: 3
                    minorVersion: 2
                }
                renderPasses: RenderPass {
                    shaderProgram: ShaderProgram {
                        vertexShaderCode:
                        loadSource("qrc:/shaders/gl3/grass.vert")
                        fragmentShaderCode:
                        loadSource("qrc:/shaders/gl3/grass.frag")
                    }
                }
            }
        ]
    }
}
```

シェーダーについてよく知らない方のために、以下の文でまとめておきます。シェーダーは、GPUによって実行されるCスタイルの構文で書かれたコンピュータプログラムです。ロジックからのデータはCPUから供給され、シェーダが実行されるGPUメモリに供給されます。ここでは、2 種類のシェーダを操作します。

* **頂点シェーダ**、メッシュのソースの各頂点で実行されます。
* **フラグメント (Fragment)** は、最終的なレンダリングを生成するために各ピクセルで実行されます。

GPU上で実行されることで、これらのシェーダはGPUの巨大な並列化パワー（CPUよりも桁違いに高い）を利用します。これにより、現代のゲームでは、このような驚異的なビジュアルレンダリングが可能になります。シェーダーとOpenGLパイプラインをカバーすることは、この本の範囲を超えています（このテーマだけで、いくつかの本棚を埋め尽くすことができます）。ここでは、Qt3Dでのシェーダの使い方を紹介することにとどめます。

***

## Info

OpenGLを深く掘り下げたい、シェーダでスキルを磨きたいという方には、グラハム・セラーズ、リチャード・S・ライト・ジュニア、ニコラス・ヘーメルの『OpenGL SuperBible』をお勧めします。

***

Qt3Dは非常に便利な方法でシェーダーをサポートしています。シェーダーファイルを.qrc リソースファイルを作成し、指定されたMaterialのeffectプロパティにロードします。

このスニペットでは、このシェーダ Technique をOpenGL 3.2上でのみ実行するように指定しています。これは、graphicsApiFilterブロックで示されています。このバージョンのOpenGLは、デスクトップマシンをターゲットにしています。デスクトップとRaspberry Piの間のパフォーマンスギャップは非常に顕著なので、プラットフォームに応じて異なるシェーダを実行できるようにしています。

というわけで、ここではRaspberry Piに対応したテクニックを紹介します。

```QML
Technique {
    graphicsApiFilter {
        api: GraphicsApiFilter.OpenGLES
        majorVersion: 2
        minorVersion: 0
    }

    renderPasses: RenderPass {
        shaderProgram: ShaderProgram {
            vertexShaderCode:
                loadSource("qrc:/shaders/es2/grass.vert")
            fragmentShaderCode:
                loadSource("qrc:/shaders/es2/grass.frag")
        }
    }
}
```

Materialのtechniquesプロパティに追加するだけです。対象となるOpenGLのバージョンはOpenGLES 2.0で、Raspberry PiやiOS/Android携帯電話でも問題なく動作することに注意してください。

最後に、パラメータをシェーダに渡す方法を説明します。ここに例を示します。

```QML
Material {
    id: material

    parameters: [
        Parameter {
            name: "score"; value: score
        }
    ]
    ...
}
```

この簡単なセクションでは、シェーダ内でscore変数にアクセスできるようになります。このMaterial要素の完全な内容については、この章のソースコードをご覧ください。草のテクスチャの上に動いて光る波を表示するシェーダを書くのが楽しくて仕方がありませんでした。

ゲームの固定要素は背景だけです。GameArea.qmlに直接追加します。

```QML
Entity {
    id: root
    ...

    Background {
        position: Qt.vector3d(camera.x, camera.y, 0)
        scale3D: Qt.vector3d(camera.x * 2, camera.y * 2, 0)
    }

    components: [frameFraph, input]
}
```

Background要素は、ヘビとリンゴの後ろの可視領域全体をカバーするように配置されます。GameArea内で定義されているため、エンティティ/コンポーネントツリーに自動的に追加され、すぐに描画されます。

***

**[戻る](../index.html)**
