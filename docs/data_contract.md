# 軌跡と transition のデータ契約

基準: [RESEARCH_SPEC.md](../RESEARCH_SPEC.md) §6・§7・§32・§38。

この契約は parser 実装前の設計案である。SCONE の実出力列、単位、刺激の記録方法は未確認。raw output との mapping を確認して schema を実装し、version を付ける。

## 形式と episode

標準形式は Parquet を初期候補とし、HDF5 も許容する。CSV は主保存形式にしない。raw output は変換後も由来を追える形で残す。

trajectory の 1 行は、1 episode の 1 観測時刻を表す。`(episode_id, time)` は一意で、time は episode 内で厳密に増加する。episode をまたぐ transition は作らない。

## 最低限の trajectory 列

| 列 | 意味 | 契約 |
| --- | --- | --- |
| `episode_id` | episode 識別子 | 空でない文字列 |
| `time` | simulation 時刻 | 秒、有限値、episode 内で単調増加 |
| `joint_positions` | 関節位置 | 固定順の数値 vector。角度/並進と単位を個別記録 |
| `joint_velocities` | 関節速度 | 対応する feature 名・単位を記録 |
| `muscle_lengths` | 筋 fiber 長 | feature 名、長さ/正規化値の区別、単位を記録 |
| `muscle_forces` | 筋力 | feature 名と単位を記録 |
| `muscle_excitations` | controller の筋刺激 | action の固定順。activation や force と混同しない |
| `contact_states` | 接触状態 | 接触対象と符号化を記録 |

Parquet の vector は feature ごとの長さを検証できる list 列を想定する。正確な Arrow 型は実装時に固定する。最低限の列が取得できない場合は integration 未完了とし、ゼロ埋めで完了扱いにしない。

追加候補は muscle fiber velocity、segment pose、center of mass、ground reaction force。利用可能性、feature 名、shape、座標系、単位を記録し、使う実験を明示する。

## `s_t, a_t, s_(t+1)` の構築

`state_features` と `action_features` に、採用列と vector の連結順を明示する。`a_t` は、その状態から次の観測までに適用された刺激として raw output と整列させる。観測ログの刺激のタイミングが異なる場合は、対応規則を記録する。

同じ episode の隣接行から transition を作り、`delta_t = time_(t+1) - time_t` を保存する。最終行は次状態がないため transition から除外し、理由を記録する。終端や simulation reset を通常の next-state に接続しない。

可変周期のまま使う場合は model に `delta_t` を渡す。固定周期へ resample する場合は周期、補間、接触状態の扱いを設定と metadata に保存する。train/test の境界をまたいで補間しない。

## metadata

`metadata.json` または dataset manifest で最低限、次を保持する。

- schema version、dataset/run/episode ID、生成時刻と timezone。
- task ID、environment ID、速度・傾斜・外乱条件、body parameter。
- optimization seed と execution seed を区別した値。
- scenario の path と hash、model と依存入力の識別・hash。
- SCONE version、simulation engine と取得可能な engine version。
- optimization fitness と、その目的関数・方向。取得できなければ `null` と理由。
- timestep、観測周期、終了理由、simulation duration、process exit code。
- feature 名、vector 順、shape、単位、座標系、欠測の規則。
- state/action の選択、raw column mapping、変換方法、データ分割。
- code commit、Python/依存 version、OS、raw/標準ファイルの所在と hash。

取得不能な値と未確認の値は区別し、推定して埋めない。SCONE version、engine、実行 seed は M1 の必須確認項目とする。

## 検証規則

- 必須列、型、vector 長、feature order が契約に一致する。
- 時刻に重複・逆転がなく、数値に説明のない NaN/Inf がない。
- state と next-state の episode が同じで、`delta_t > 0` である。
- stimulus と state の時間対応を検証できる。
- model が定める筋刺激の範囲と contact の符号化に従う。
- 正規化の統計は訓練集合のみから計算し、評価時は固定する。
- 欠測、除外行、終端、resample を記録し、変換失敗を隠さない。

状態・行動の違う身体モデルを共同学習する場合は、feature 対応と不足 feature の扱いを別の契約で定める。位置が同じ vector だから意味も同じとはみなさない。
