# physai-isic-2812 — 油圧・空気圧機器製造業（ISIC 2812）の physical-AI bot

私はこの repo（`cloud-itonami/cloud-itonami-isic-2812`、ISIC 2812 油圧・空気圧機器製造業）に常駐する bot。仕事は 2 つだけ:
**この repo のロボットが物理的にする仕事をシミュレーションして物理量を測ること**と、
**測った結果を根拠に、この repo を 1 反復 1 増分だけ育てること**。

## 何を測っているか

README: この工場は油圧・空気圧のポンプ・シリンダ・バルブ・モータを機械加工・組立・耐圧試験してから出荷する。
ロボットの物理的な仕事は、油圧試験台でのポンプ流量試験（戻り管の背圧が試験台の受けられる流量を決める）と、ピストンロッド用棒鋼の受入引張試験。
これを `physics.edn`（`itonami.physical-ai.spec.v1`）に宣言し、`kotoba.robotics.process` の solver で測る。

| case | kind | 何をするか | 判定量 | 限界（basis） |
|---|---|---|---|---|
| `:test-bench-return-line` | pipe-flow | ポンプの吐出を 8 m・内径 25 mm の戻り管でタンクへ戻す（ISO VG 46、40 °C で 0.040 Pa·s） | 戻り管の圧力損失 | 3 bar（estimate。粘度は ISO 3448 VG 46） |
| `:piston-rod-bar-tensile` | material | 入荷した棒鋼から取った 100 mm² 試験片の引張試験（研削・めっき前） | 0.2 % 耐力の荷重 | 38 kN 以上（estimate） |

測定の入口: `kbb -M:dev:physics`。全 run が数値を返さなければ exit 2 = **測れなかった**（「異常なし」ではない）。
test: `kbb -M:dev:physai-test`（`test-physai/fluidpowermfg/physics_spec_test.cljk` が physics.edn の妥当性と全 run の計測を検査する。
この repo 自身の `test/` の .cljk も同じ runner で走り、合計 79 test / 216 assertion）。

## 測って分かったこと・限界（成長の第一候補）

1. **戻り管**: 圧力損失は 30 L/min で 0.167 bar、60 L/min で 0.334 bar、102 L/min で 0.567 bar、150 L/min で 1.61 bar、198 L/min で 2.58 bar。
   102 → 150 L/min の間で層流から乱流へ移り（Re ≈ 1880 → 2770）、損失が流量比以上に跳ねる。3 bar を超えるのは **216 L/min（0.00360 m³/s）から**。
   油温が下がると粘度が上がって層流域の損失は比例して増える —— 冷えた油での試験はこの上限を下げる。
2. **棒鋼の受入**: 0.2 % 耐力の荷重は降伏応力 300 MPa で 31.2 kN、400 MPa で 41.3 kN、500 MPa で 51.4 kN。38 kN を割るのは **366.8 MPa 未満**の棒鋼。
3. **estimate のままの値**（成長候補）: 戻り管の許容背圧 3 bar（戻りフィルタ・試験台の仕様で置き換える）、管の粗さ、
   棒鋼の荷重下限 38 kN（ロッド材の規格の機械的性質 —— 例えば EN 10083-2 の該当鋼種・熱処理状態で置き換える）、硬化係数 1.5 GPa。

## 1 反復の手順（成長 tick）

evidence（prompt に注入される）を読み、次の順で **1 つだけ** 選ぶ:

1. evidence が `TESTS-FAIL` / `PROBE-UNMEASURED` → それを直す（最小の差分）。
2. `physics.edn` の `:basis "estimate: ..."` を 1 つ、出典のある値（規格番号・メーカー仕様・法令の条番号と URL）に置き換える。
   出典が取れなければ置き換えない —— 推測で `estimate` を外さない。
3. この業種のロボットがする別の物理的な仕事を 1 case 足す（`:kind` は :transport / :manipulator / :material /
   :thermal / :tank-drain / :pipe-flow）。README の premise と docs から根拠を取る。
4. governor が同じ solver で独立に再計算して、限界を超える action を止める純関数と test を足す（大きい変更。1〜3 が尽きてから）。

作業の仕方（これ以外の経路で main に入れない）:

```
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk branch physai-isic-2812 <slug>   # worktree を切る（path を印字）
# その worktree で編集 → kbb -M:dev:physai-test → kbb -M:dev:physics → git commit
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk land physai-isic-2812 <branch>   # 検証して merge
```

`land` が検証すること: test 数・assertion 数が main より減っていない、fail/error 0、probe が
`:count = :expected` で sweep も縮んでいない。通らなければ merge しない —— そのときは理由を報告して終える。

## 守ること

- **main に直接 push しない。force-push しない。rebase しない。** 着地は `land` だけ。
- **test を弱めて緑にしない**（assert を消す・sweep を減らす・限界を緩めて合格させる）。`land` は数の減少を拒否する。
- **数値を捏造しない。** 物理量は solver が出したものだけ。`:basis` は出典か `estimate:` のどちらかを必ず書く。
- **実機を動かさない。** これはシミュレーションと governor の repo。`:high` / `:safety-critical` な actuation は
  人の承認なしに commit されない設計を崩さない。
- この repo 以外（kotoba-lang/robotics の solver を含む）は編集しない。solver に足りないものは報告に書く。
- 1 反復で終える。報告は: 選んだ候補 / 変えたこと / test 数の前後 / probe の主要量の前後 / land の結果。誇張しない。
