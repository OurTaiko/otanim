# otanim

轻量 2D 动画格式 `.otanim`，以及围绕它的工具。

- [SPEC.md](SPEC.md)：格式规范（版本 1）和 puchichara 的片段名约定。
- 根目录的 Go 包 `ourtaiko.org/otanim`（`otanim.go`）：读写的参考实现。
- `spine/`：Spine 3.8 二进制骨骼与图集的读取和姿势求解。按公开的格式自己写，没有用官方 Spine Runtimes（它的许可要求 Spine 授权）。
- `cmd/spine2otanim`：Spine → otanim 转换器。
- `cmd/skeldump`：打印一个 `.skel` 用到了什么（约束、附件种类、时间轴），用来判断转换器支不支持。

```sh
go test ./...
```

## 谁在用

| 地方 | 做什么 |
|---|---|
| OurTaikoPlay `Assets/OurTaiko/Runtime/Chara/Otanim.cs`、`PuchiAnimGraphic.cs` | 游戏里读取和播放（C#） |
| OurTaikoPlay `Assets/OurTaiko/Tests/Shared/OtanimFixture.cs` | 按 SPEC.md 写测试用的小文件 |
| OurTaikoNet `web/src/kit/don3d/otanim.ts` | 网站换装页和 3D 预览里读取和播放（TypeScript） |
| OurTaikoNet `tools/3dchara/catalog.py` | 公开桶 `3dchara/puchi/<id>.otanim` 列进目录 |

改格式时这几处要一起改。

## spine2otanim

```sh
go run ./cmd/spine2otanim -list 角色.skel                      # 列出动画和皮肤
go run ./cmd/spine2otanim -init 角色.json 角色.skel            # 生成配置模板
go run ./cmd/spine2otanim -config 角色.json                    # 按配置转换
go run ./cmd/spine2otanim -clips "idle=Idle,gogo=Skill_3_Begin+Skill_3_Loop,happy=Skill_1:once" -max-texture 512 -preview /tmp/preview 角色.skel
go install ./cmd/spine2otanim                                  # 装成命令
```

- 读 Spine **3.8 二进制** `.skel`（同名 `.atlas` 与页 PNG 放在一起）；JSON 与 4.x 还不读。
- 支持：骨骼（五种变换模式）、区域／网格／带权网格／链接网格、IK（含 softness、stretch、compress）、变换约束（四种）、全部 3.8 时间轴（事件不需要）、绘制顺序、皮肤（`-skin`）。**不支持**：路径约束（报错）、裁剪附件（警告，按不裁剪画）、双色（tint black，警告，忽略暗色）。
- `-clips`：`名字=动画`，`:once` 为一次性；`名字=A+B` 把几段动画首尾相接成一个片段，循环时最后一段循环、前面的只播一次（写成循环起点）。不写就把所有有时长的动画按原名做成循环片段。
- **配置文件**（推荐）：`-init 角色.json 骨骼.skel` 按骨骼里的动画生成模板（每段动画一个片段，`comment` 里写了游戏认的片段名），改好后 `-config 角色.json` 转换。路径相对于配置文件；同时给的命令行参数优先。字段：`skeleton`、`atlas`、`output`、`name`、`skin`、`fps`、`unit`、`pad`、`view`、`pma`、`maxTexture`、`keepTexture`、`preview`、`previewSize`、`icon`、`iconSize`、`clips`（`[{"name","play","loop"}]`，`play` 是动画名、`"A+B"` 或数组，`loop` 默认 true）。拼错的字段名会报错。
- 贴图默认只保留用到的部分重新打包（`-keep-texture` 原样），并缩到 `-max-texture`（默认 1024）以内；预乘自动判断（`-pma`）。
- 视野默认是所有片段的包围盒加 `-pad`（默认 4）；`-view` 手动指定（角色在不同片段里走出很远时用）。
- `-preview <目录>`：把写好的文件读回来，每个片段画一张帧条 PNG 和一段 GIF，用来检查。

- `-icon <文件>`（配置里 `icon`）：把 `idle` 片段（没有就第一个）的第一帧按它自己的范围取景，画成 `-icon-size`（默认 128）见方的透明 PNG，给换装页的列表当图标（放到桶里的 `icons/puchi/<id>.png`）。

## 例子

例：一个明日方舟角色（维什戴尔）的战斗正面小人，配置为 `idle=Idle`、`gogo=Skill_3_Begin+Skill_3_Loop`（第 25 帧起循环）、`combo=Attack_A:once`、`happy=Skill_1:once`、`miss=Stun`，转出来 167 KB。素材版权归原作者，不在本仓库里。

例：同一角色的基建小人（Spine 3.8.99，47 骨骼、55 网格），只取 Relax／Move／Interact／Sleep 四段，贴图缩到 512：**186 KB**（贴图 166 KB，四段动画合计约 20 KB）。全部 7 段、原图集：978 KB。

