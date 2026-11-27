# 朝の秘書（スケジュールタスク）の指示文

## これは何か

- 毎朝7時ごろ、Notionを読んで「今日の一枚」をDiscordの専用チャンネルに送る、スケジュールタスク（シオン新聞と同じ仕組み）の指示文。
- スケジュールタスク「今日の一枚」（毎日 6:54・日本時間）が、毎回このファイルをリポジトリから読んで動く。だから、このファイルを直してpushすれば、次の朝から新しい手順で動く。
- Discordの投稿用URL（Webhook URL）は、スケジュールタスクの設定の中だけにある。このファイルには `{{DISCORD_WEBHOOK_URL}}` とだけ書く。
- 判定と文面は、下のスクリプトが作る。だから毎朝、同じルール・同じ形で届く（AIの気分でぶれない）。
- 2026-11-26に、このスクリプトを実際のNotionのデータで試した（出力例は、個人の情報が入らないよう作った例のデータで出したもの）。

## 決めたこと

| 項目 | 決めたこと |
|---|---|
| 読み方 | Notionのビューを読む（無料プランでも回数無制限）。SQLは使わない |
| 数式の列 | 使わない（Notion MCPでは値が読めないため）。元の列からスクリプトが判定する |
| やる日の意味 | **始めたい日**。この日から候補に出す。過ぎても失敗とは見なさない（🟠持ち越しの印はやめた。2026-11-26） |
| 急ぎの印 | 期限だけ。🔴期限切れ／⏰今日まで・明日まで・あとN日（3日以内） |
| 書き込み | ①選んだタスクのうちやる日が空のものに、今日の日付を入れる ②✓（済）が付いているのに完了になっていないタスクを、完了にする。ほかの列は変えない。消さない |
| 代行 | しない（無人で動くため） |
| 失敗したとき | Notionが読めなければ「Claudeアプリで『今日なにやる？』と聞いて」と送る |

---

## 指示文（ここから下を、スケジュールタスクのプロンプトにする）

これは、毎朝自動で動くスケジュールタスク。過去の会話は見られない。途中で質問や確認をせず、下の手順だけで最後まで実行する。

あなたはシオンの朝の秘書。Notionの「仕事管理」を読み、今日の一枚をDiscordに送る。

### 守ること

- 判定と文面は、下のスクリプトが作る。スクリプトの出力を書き換えたり、足したりしない。
- Notionに書き込むのは2つだけ：スクリプトが選んだタスクの「やる日」と、スクリプトが完了にすると決めたタスクの「状態・完了日・次の一歩」。ほかの列は変えない。消さない。
- Notionは、ビューを読む方法（notion-query-data-sources の mode "view"）だけで読む。SQLやrowsモードは使わない（無料プランの回数上限があるため）。
- 数式・集計の列（サイン・集計など）は値ではなく「formulaResult://…」という参照が返る。これで正常。使わない。
- 固定のパス（/tmp/xxx など）は使わない。作業場所は毎回 mktemp -d で作る。

### 手順

1. 作業場所と日付を用意する（bash）：

   ```bash
   D=$(mktemp -d) && cd "$D" && TZ=Asia/Tokyo date '+%F %-j'
   ```

   出力の1つ目が今日の日付（YYYY-MM-DD）、2つ目が通し日（1〜366）。以降のファイルはすべて $D に置く。

2. Notionを読む（Notionコネクタの notion-query-data-sources）：
   - タスク：`{"data":{"mode":"view","view_url":"view://4237db70-148e-81c9-b91e-000c24950f81","page_size":100}}`
     has_more が true のときは、start_cursor に next_cursor を入れて続きを読み、全部つなげる。
   - プロジェクト：`{"data":{"mode":"view","view_url":"view://4237db70-148e-815b-85be-000cd955e3bb","page_size":100}}`
   - 失敗したら1回だけやり直す。それでも失敗したら、手順7を行って終わる。

3. 読んだ results の配列を、$D/tasks.json と $D/projects.json に書き出す（ヒアドキュメント）。
   - 各行の「名前・url・状態・プロジェクト・次の一歩・済・date:やる日:start・date:期限:start・date:完了日:start・担当・目安時間・最終更新」は、値を変えずにそのまま書く（ほかの列は省いてよい）。
   - 列の名前はNotionで変えられることがある。タイトルの列が「タスク」、チェックの列が「✓」や名前なし（`""`）になっていても、**読んだときの名前のまま**書く（スクリプトが読み替える）。
   - プロジェクトは「名前・url・状態・領域・date:開始日:start・date:期限:start」をそのまま書く。
   - 書き出したら件数を確かめる：`python3 -c "import json;print(len(json.load(open('tasks.json'))),len(json.load(open('projects.json'))))"`。Notionから読んだ件数と同じであること。

4. 下の「スクリプト」を、そのまま $D/brief.py に書き出して実行する：

   ```bash
   python3 brief.py <今日の日付> <通し日> tasks.json projects.json
   ```

   _body.txt（送る本文）、_new_picks.json（やる日を入れるタスクのURL）、_complete.json（完了にするタスク）ができる。出力の最後の行の CHARS が1900を超えていたら、送らずに手順8で報告する。

5. Notionに書き込む（notion-update-page、command: "update_properties"）。失敗したものは飛ばして続ける（送信は止めない）。
   - _new_picks.json の各URLに：

     ```json
     {"date:やる日:start": "<今日の日付>", "date:やる日:is_datetime": 0}
     ```

   - _complete.json の各項目（url と date）に：

     ```json
     {"状態": "完了", "date:完了日:start": "<その項目のdate>", "date:完了日:is_datetime": 0, "次の一歩": "__NO__"}
     ```

6. Discordに送る（_body.txt をそのまま送る）：

   ```bash
   python3 -c "import json;json.dump({'content':open('_body.txt',encoding='utf-8').read()},open('_p.json','w',encoding='utf-8'),ensure_ascii=False)"
   curl -s -w "\nHTTP:%{http_code}\n" -X POST -H "Content-Type: application/json" --data-binary @_p.json "{{DISCORD_WEBHOOK_URL}}"
   ```

   - python3のurllibでは送らない（Cloudflareに弾かれる）。必ずcurlを使う。
   - HTTP:204 なら成功。それ以外なら、ステータスと応答を見て原因を直し、1回だけ送り直す。同じ内容を2回送らない。

7. Notionが読めなかったとき：_body.txt を次の内容で作り（M/D(曜) は今日の日付）、手順6と同じ方法で送る。

   ```
   ☀️ 今日の一枚　M/D(曜)
   今朝はNotionを読めませんでした。
   Claudeアプリで「今日なにやる？」と聞いてください。
   ```

8. 最後の報告（3行以内）：送信結果（HTTPコード）／今日の一枚の件数／やる日を書き込んだ件数と、完了にした件数（失敗があればタスク名）。本文はチャットに貼らない。

### スクリプト

```python
# -*- coding: utf-8 -*-
# 今日の一枚を作る。使い方: python3 brief.py YYYY-MM-DD 通し日(1-366) tasks.json projects.json
# 出力: _body.txt（Discordに送る本文）、_new_picks.json（やる日を今日にするタスクのURL）、
#       _complete.json（✓が付いていたので完了にするタスク）
import json, sys, re, datetime as dt

TODAY = dt.date.fromisoformat(sys.argv[1])
DOY = int(sys.argv[2])
def load(path):
    x = json.load(open(path, encoding='utf-8'))
    return x.get('results', []) if isinstance(x, dict) else x
tasks, projects = load(sys.argv[3]), load(sys.argv[4])
# 列の名前がNotionで変えられても読めるようにする（タイトル＝名前／タスク、チェック＝済／✓／名前なし）
for t in tasks:
    if '名前' not in t:
        t['名前'] = t.get('タスク') or t.get('タイトル') or '（名前なし）'
    if '済' not in t:
        for k in ('✓', '✅', ''):
            if k in t:
                t['済'] = t[k]
                break

JST = dt.timezone(dt.timedelta(hours=9))
WD = '月火水木金土日'
MIN = {'15分': 15, '30分': 30, '1時間': 60, '2時間以上': 120}
HOME = 'https://app.notion.com/p/4237db70148e812197b1e20e3cc80756'
FAR = dt.date(9999, 12, 31)
SOON = 3          # 期限が何日以内なら「近い」とするか
STALE = 14        # 始めたい日から何日たったら「まだやる？」と聞くか

def key(u):
    return re.sub(r'[^0-9a-f]', '', (u or '').lower())[-32:]
def day(r, name):
    v = (r or {}).get(f'date:{name}:start')
    return dt.date.fromisoformat(v[:10]) if v else None
def edited(r):
    v = r.get('最終更新')
    if not v:
        return TODAY
    return dt.datetime.fromisoformat(v.replace('Z', '+00:00')).astimezone(JST).date()
def md(d):
    return f'{d.month}/{d.day}'
def hm(n):
    h, m = divmod(n, 60)
    return ((f'{h}時間' if h else '') + (f'{m}分' if m else '')) or '0分'

# ── ✓（済）が付いているのに、まだ完了になっていないもの → 完了にする ──
complete = []
for t in tasks:
    if t.get('済') == '__YES__' and t.get('状態') != '完了':
        d = min(edited(t), TODAY)
        complete.append({'url': t['url'], 'name': t['名前'], 'date': d.isoformat()})
        t['状態'] = '完了'          # 以降は完了として数える
        t['次の一歩'] = '__NO__'

P = {key(p['url']): p for p in projects}
for t in tasks:
    try:
        rel = json.loads(t.get('プロジェクト') or '[]')
    except Exception:
        rel = []
    p = P.get(key(rel[0])) if rel else None
    st = t.get('状態')
    t['_pkey'] = key(p['url']) if p else ''
    t['_pname'] = p['名前'] if p else ''
    t['_pdue'] = day(p, '期限') or FAR
    t['_due'] = day(t, '期限')
    t['_do'] = day(t, 'やる日')
    t['_idle'] = (TODAY - edited(t)).days
    t['_open'] = st in ('これから', '進行中', '待ち')
    t['_left'] = (t['_due'] - TODAY).days if (t['_open'] and t['_due']) else None   # 期限まであと何日（マイナス＝過ぎた）
    t['_mine'] = t.get('担当') != 'Claude'
    t['_next'] = t.get('次の一歩') == '__YES__'

active = [p for p in projects if p.get('状態') == '進行中']
active_keys = {key(p['url']) for p in active}

# ── ☀️ 今日の候補（シオンのタスクだけ・最大5件）──
# やる日＝始めたい日。過ぎても失敗ではない。やる日が先のものは、まだ出さない。
cand = [t for t in tasks if t.get('状態') in ('これから', '進行中') and t['_mine']
        and (t['_do'] is None or t['_do'] <= TODAY)]
g1 = sorted([t for t in cand if t['_left'] is not None and t['_left'] <= SOON], key=lambda t: t['_due'])     # ①期限が3日以内（過ぎたものも）
g2 = sorted([t for t in cand if t['_do'] is not None], key=lambda t: (t['_do'], t['_pdue'], t['名前']))     # ②始めたい日が来ている（古い順）
g3 = sorted([t for t in cand if t['_do'] is None and t['_next'] and t['_pkey'] in active_keys],
            key=lambda t: (t['_pdue'], t['_pname']))                                                     # ③次の一歩
picked, seen = [], set()
for t in g1 + g2 + g3:
    if t['url'] in seen:
        continue
    seen.add(t['url'])
    if len(picked) < 5:
        picked.append(t)
inlist = {t['url'] for t in picked}
overflow = len({t['url'] for t in g1 + g2} - inlist)
new_picks = [t['url'] for t in picked if t['_do'] is None]   # やる日が空のものだけ、今日を入れる

def sign(t):
    n = t['_left']
    if n is None or n > SOON:
        return ''
    if n < 0:
        return f"🔴期限{md(t['_due'])}"
    return {0: '⏰今日まで', 1: '⏰明日まで'}.get(n, f'⏰あと{n}日')

L = [f"☀️ 今日の一枚　{md(TODAY)}({WD[TODAY.weekday()]})", '━━━━━━━━━━━━━━']
if picked:
    L.append('上からやれたら十分。残りは明日も候補に出ます')
    for t in picked:
        m = t.get('目安時間')
        L.append(f"□ {t['名前']}" + (f"（{m}）" if m else ''))
        sub = '・'.join(x for x in [sign(t), t['_pname']] if x)
        if sub:
            L.append(f'　{sub}')
    ms = [MIN[t['目安時間']] for t in picked if t.get('目安時間') in MIN]
    if ms:
        more = any(t.get('目安時間') == '2時間以上' for t in picked)
        L.append(f"全部やると約{hm(sum(ms))}" + ('以上' if more else ''))
else:
    L += ['□ 今日の候補はありません', '　Claudeに「今日なにやる？」と話すと、一緒に選べます']

if complete:
    names = '、'.join(c['name'] for c in complete[:3]) + (f' ほか{len(complete) - 3}件' if len(complete) > 3 else '')
    L += ['', f'✅ ✓の付いた{len(complete)}件を完了にしました（{names}）']

# ── ❓ これやった？（最大3件・今日の候補に出したものは除く）──
q = []
for t in sorted([t for t in tasks if t.get('状態') == '待ち' and t['_idle'] >= 7 and t['url'] not in inlist], key=lambda t: -t['_idle']):
    q.append([f"・{t['名前']}", f"　待ち{t['_idle']}日目。返事きた？"])
for t in sorted([t for t in tasks if t.get('状態') == '進行中' and t['_idle'] >= 3 and t['url'] not in inlist], key=lambda t: -t['_idle']):
    q.append([f"・{t['名前']}", f"　進行中のまま{t['_idle']}日。終わった？"])
ai = sorted([t for t in tasks if t.get('状態') == 'これから' and not t['_mine']], key=lambda t: (t['_pdue'], t['名前']))
if ai:
    t = ai[DOY % len(ai)]
    q.append([f"・🤖 {re.sub(r'^🤖\s*', '', t['名前'])}", '　Claudeが調べられます。方針を見て任せる？'])
stale = sorted([t for t in cand if t['_do'] is not None and (TODAY - t['_do']).days >= STALE and t['url'] not in inlist],
               key=lambda t: (t['_do'], t['名前']))
if stale:
    t = stale[DOY % len(stale)]
    q.append([f"・{t['名前']}", f"　始めたい日から{(TODAY - t['_do']).days}日。まだやる？分ける？"])
ideas = sorted([t for t in tasks if t.get('状態') == 'アイデア' and t['_idle'] >= 14], key=lambda t: (-t['_idle'], t['名前']))
if ideas:
    t = ideas[DOY % len(ideas)]
    q.append([f"・💡 {t['名前']}", f"　{t['_idle']}日前のアイデア。やる？やめる？"])
q = q[:3]
if q:
    L += ['', '❓ これやった？'] + [x for it in q for x in it]

# ── 📌 ほかに（入りきらなかった件数・プロジェクトの気になる点）──
pin = []
if overflow:
    pin.append(f'・ほかに候補が{overflow}件（Notionの「☀️ 今日」「🗂 ボード」で見られます）')
for p in sorted(active, key=lambda p: (day(p, '期限') or FAR, p['名前'])):
    k = key(p['url'])
    mine = [t for t in tasks if t['_pkey'] == k]
    real = [t for t in mine if t.get('状態') != 'アイデア']
    done = [t for t in real if t.get('状態') == '完了']
    pd, ps = day(p, '期限'), day(p, '開始日')
    if pd and pd < TODAY:
        pin.append(f"・「{p['名前']}」の期限（{md(pd)}）を過ぎています。延ばす？")
    elif pd and ps and (pd - ps).days > 0 and real:
        el, pr = (TODAY - ps).days / (pd - ps).days, len(done) / len(real)
        if el - pr >= 0.3:
            pin.append(f"・「{p['名前']}」が遅れ気味（期間の{round(el * 100)}%・進み{round(pr * 100)}%）")
    if mine and min(t['_idle'] for t in mine) >= 7:
        pin.append(f"・「{p['名前']}」が{min(t['_idle'] for t in mine)}日動いていません")
    if not any(t['_next'] and t['_open'] for t in mine):
        pin.append(f"・「{p['名前']}」の次の一歩を決めよう（Claudeに話すと案が出ます）")
if len(pin) > 5:
    pin = pin[:4] + [f'・ほか{len(pin) - 4}件']
if pin:
    L += ['', '📌 ほかに'] + pin

L += ['', '終わったら「☀️ 今日」で✓。「〇〇終わった」とClaudeに言ってもOK', f'📒 <{HOME}>']
body = '\n'.join(L)
open('_body.txt', 'w', encoding='utf-8').write(body)
json.dump(new_picks, open('_new_picks.json', 'w', encoding='utf-8'), ensure_ascii=False)
json.dump(complete, open('_complete.json', 'w', encoding='utf-8'), ensure_ascii=False)
print(body)
print('---')
print('NEW_PICKS', json.dumps(new_picks, ensure_ascii=False))
print('COMPLETE', json.dumps([c['url'] for c in complete], ensure_ascii=False))
print('CHARS', len(body))
```

## 指示文ここまで

---

## 選び方（スクリプトがやっていること）

| 順 | 候補 | 並び |
|---|---|---|
| ① | 期限が3日以内のもの（過ぎたものも） | 期限が近い順 |
| ② | 始めたい日（やる日）が来ているもの | やる日が古い順 |
| ③ | やる日が空の、進行中プロジェクトの次の一歩 | プロジェクトの期限が近い順 |

- シオンのタスクだけ（担当Claudeと待ちは入れない）。✓が付いたもの、やる日が先のものは入れない。上から5件まで。
- 入りきらなかった件数は 📌 に出す。
- ❓（3件まで・今日の候補に出したものは除く）：①待ちのまま7日以上 ②進行中のまま3日以上 ③🤖Claudeに任せられるもの（1日1件・日替わり） ④始めたい日から14日以上たったもの（1日1件・日替わり。「まだやる？分ける？」） ⑤14日以上前のアイデア（1日1件・日替わり）。

## 出力例（作った例のデータで、11/27として実行）

```
☀️ 今日の一枚　11/27(金)
━━━━━━━━━━━━━━
上からやれたら十分。残りは明日も候補に出ます
□ 種籾の手配先と注文の締切を確かめる（15分）
　⏰明日まで・来年の米作り計画
□ 本を1か所に集める（30分）
　自室の片付け
□ 不燃ごみを捨てに行く（2時間以上）
　納屋の片付け
全部やると約2時間45分以上

✅ ✓の付いた1件を完了にしました（来年使う田んぼの名前を紙に書き出す）

❓ これやった？
・金属ゴミを1か所に集める
　進行中のまま4日。終わった？
・🤖 入れ物の候補を3件探す
　Claudeが調べられます。方針を見て任せる？

📌 ほかに
・「来年の米作り計画」の次の一歩を決めよう（Claudeに話すと案が出ます）

終わったら「☀️ 今日」で✓。「〇〇終わった」とClaudeに言ってもOK
📒 <https://app.notion.com/p/4237db70148e812197b1e20e3cc80756>
```

- 「本を1か所に集める」は、始めたい日（11/26）を過ぎているが、印は付かない（失敗ではないため）。
- 「田んぼの名前」は前の日に✓が付いていたので、完了にしてから数えた。次の一歩がなくなったので 📌 に出た。
- 429文字。Discordの上限（2000文字）まで余裕がある。

## 紙への写し方

各タスクの**1行目（□の行）だけ**を紙に写す。2行目は補足（印とプロジェクト名）なので写さなくてよい。
