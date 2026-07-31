# TrainWatcher 追跡パネル 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** TrainWatcher の ALL モードに、通勤経路4駅の時刻表を現在時刻基準で表示する追跡パネルを追加する。

**Architecture:** Yahoo!路線情報から時刻表を Python スクリプトで一括取得し、数値配列として `index.html` に静的埋め込みする。表示側は既存コードに触れない独立した関数群として実装し、ALL モード時のみ描画する。

**Tech Stack:** Python 3.11（requests / BeautifulSoup4 / unittest）、素の HTML+CSS+JavaScript（フレームワークなし・単一ファイル）

**設計書:** `work/OLDproject/TrainWatch/SPEC_tracker.md`

---

## Global Constraints

以下は全タスクに暗黙的に適用される。

- **git の変更系コマンドは実行しない。** `commit` / `push` / `reset` などはコマンドを提示するだけで、実行はユーザーが行う（CLAUDE.md ルール2）
- **`index.html` の文字コードは UTF-8(BOMなし)、改行は CRLF を維持する。** 編集のたびにバイト単位で検証する（CLAUDE.md ルール3）
- **既存挙動を壊さないことが最優先。** 天気・カウントダウン・電車カード・ランクサマリー・モード切替の挙動は改修前と完全同一であること（CLAUDE.md ルール11）
- **コード内コメントは1行で簡潔に。** 経緯や調査根拠は `SPEC_tracker.md` に書く（CLAUDE.md ルール4）
- **pytest は未インストール。** テストは標準ライブラリの `unittest` で書き、`python -m unittest` で実行する
- **node/bun/deno は存在しない。** JS のロジック検証はブラウザで開く `_selftest.html` で行う
- 作業用スクリプトは scratchpad 配下に置き、成果物には含めない。配布物は `index.html` 単体のまま
- スクレイピングは 1リクエストあたり 0.6秒以上の間隔を空け、取得済みレスポンスはディスクキャッシュして再取得しない
- User-Agent は `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126 Safari/537.36` を使う

### パス定義

以下を各タスクで参照する。

- `PROJ` = `C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch`
- `WORK` = `C:\Users\SHOTAT~1\AppData\Local\Temp\claude\C--Users-ShotaTanaka-Documents--------\7786d944-750b-4f47-bc1a-27177caa3590\scratchpad\tw_build`

---

## File Structure

| ファイル | 責務 |
|---|---|
| `WORK/boards.py` | 8ボードの定義（駅ID・路線方面ID・到着駅・取得方法）のみを持つ定数モジュール |
| `WORK/yahoo_fetch.py` | HTTP 取得とディスクキャッシュ。パースは一切しない |
| `WORK/parse_index.py` | 時刻表インデックスページの HTML → 列車リスト＋凡例 |
| `WORK/parse_detail.py` | 列車詳細ページの HTML → 停車駅ごとの着発時刻 |
| `WORK/build.py` | 上記を束ねてデータを組み立て、JS の `const` 宣言テキストを出力 |
| `WORK/holidays.py` | 祝日＋年末年始リストを生成 |
| `WORK/verify.py` | 生成データの整合性チェックとレポート出力 |
| `WORK/tests/*.py` | 各パーサ・ビルダの unittest |
| `WORK/fixtures/*.html` | テスト用に固定した実 HTML |
| `WORK/cache/` | HTTP レスポンスキャッシュ |
| `WORK/out/timetable.js` | 生成された JS データ（`index.html` へ貼る元） |
| `PROJ/index.html` | 追跡パネルの CSS・DOM・JS を追加 |
| `PROJ/_selftest.html` | JS ロジック検証用。**最後に削除する** |
| `PROJ/SPEC_tracker.md` | 設計書（作成済み） |

---

## Task 1: ボード定義とHTTP取得基盤

**Files:**
- Create: `WORK/boards.py`
- Create: `WORK/yahoo_fetch.py`
- Create: `WORK/tests/test_yahoo_fetch.py`

**Interfaces:**
- Consumes: なし
- Produces:
  - `boards.BOARDS` — `list[Board]`。`Board` は `dataclass(id, dir, station, line, station_id, line_id, arr_station, arr_label, mode)`。`dir` は `'go'|'back'`、`mode` は `'detail'|'fixed'`
  - `yahoo_fetch.fetch(path: str) -> str` — `path` は `/timetable/...` 形式。UTF-8 デコード済み HTML を返す。キャッシュヒット時は即返す
  - `yahoo_fetch.CACHE_DIR` — `pathlib.Path`

- [ ] **Step 1: `WORK/boards.py` を作成**

```python
"""8ボードの定義。設計書 SPEC_tracker.md セクション3.1 に対応。"""
from dataclasses import dataclass


@dataclass(frozen=True)
class Board:
    id: str           # 'G1' 等
    dir: str          # 'go' | 'back'
    station: str      # 駅ボタンのラベル
    line: str         # 遅延マーク用。TARGET_LINES の名称と一致させる
    station_id: int
    line_id: int
    arr_station: str  # 到着列に出す駅名。'' なら終着駅
    arr_label: str    # 到着列の見出し略記
    mode: str         # 'detail' = 詳細ページ実取得 / 'fixed' = 固定所要時間


BOARDS = [
    Board('G1', 'go',   '秦野',     '小田急小田原線', 23283, 3090, '相模大野', '大野', 'detail'),
    Board('G2', 'go',   '相模大野', '小田急江ノ島線', 23172, 3111, '中央林間', '林間', 'fixed'),
    Board('G3', 'go',   '中央林間', '東急田園都市線', 23231, 3160, '南町田グランベリーパーク', '南町田', 'fixed'),
    Board('G4', 'go',   '南町田',   '東急田園都市線', 22998, 3160, '', '終着', 'detail'),
    Board('K1', 'back', '南町田',   '東急田園都市線', 22998, 3161, '中央林間', '林間', 'fixed'),
    Board('K2', 'back', '中央林間', '小田急江ノ島線', 23231, 3110, '相模大野', '大野', 'fixed'),
    Board('K3', 'back', '相模大野', '小田急小田原線', 23172, 3091, '秦野', '秦野', 'detail'),
    Board('K4', 'back', '秦野',     '小田急小田原線', 23283, 3091, '', '終着', 'detail'),
]

# Yahoo のダイヤ区分パラメータ
KINDS = {'wd': 1, 'sa': 2, 'ho': 4}

BOARD_BY_ID = {b.id: b for b in BOARDS}
```

- [ ] **Step 2: 失敗するテストを書く**

`WORK/tests/test_yahoo_fetch.py`:

```python
import shutil
import unittest
from pathlib import Path

import sys
sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

import yahoo_fetch
import boards


class TestBoards(unittest.TestCase):
    def test_eight_boards(self):
        self.assertEqual(len(boards.BOARDS), 8)

    def test_ids_unique(self):
        ids = [b.id for b in boards.BOARDS]
        self.assertEqual(len(ids), len(set(ids)))

    def test_four_go_four_back(self):
        self.assertEqual(sum(1 for b in boards.BOARDS if b.dir == 'go'), 4)
        self.assertEqual(sum(1 for b in boards.BOARDS if b.dir == 'back'), 4)

    def test_detail_boards_are_the_long_legs(self):
        detail = sorted(b.id for b in boards.BOARDS if b.mode == 'detail')
        self.assertEqual(detail, ['G1', 'G4', 'K3', 'K4'])

    def test_lines_match_target_lines(self):
        # index.html の TARGET_LINES に存在する名称であること
        allowed = {'小田急小田原線', '小田急江ノ島線', '東急田園都市線'}
        for b in boards.BOARDS:
            self.assertIn(b.line, allowed, b.id)


class TestFetch(unittest.TestCase):
    def setUp(self):
        self.tmp = Path(__file__).parent / '_tmpcache'
        if self.tmp.exists():
            shutil.rmtree(self.tmp)
        yahoo_fetch.CACHE_DIR = self.tmp

    def tearDown(self):
        if self.tmp.exists():
            shutil.rmtree(self.tmp)

    def test_fetch_caches_to_disk(self):
        path = '/timetable/23283/3090?kind=1'
        first = yahoo_fetch.fetch(path)
        self.assertIn('hh_8', first)
        files = list(self.tmp.glob('*.html'))
        self.assertEqual(len(files), 1)

    def test_second_fetch_hits_cache(self):
        path = '/timetable/23283/3090?kind=1'
        yahoo_fetch.fetch(path)
        yahoo_fetch.NETWORK_CALLS = 0
        yahoo_fetch.fetch(path)
        self.assertEqual(yahoo_fetch.NETWORK_CALLS, 0)


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 3: テストを実行して失敗を確認**

```bash
cd "$WORK" && python -m unittest discover -s tests -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'yahoo_fetch'`

- [ ] **Step 4: `WORK/yahoo_fetch.py` を実装**

```python
"""Yahoo!路線情報の HTTP 取得とディスクキャッシュ。"""
import hashlib
import time
from pathlib import Path

import requests

BASE = 'https://transit.yahoo.co.jp'
UA = ('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 '
      '(KHTML, like Gecko) Chrome/126 Safari/537.36')
CACHE_DIR = Path(__file__).parent / 'cache'
INTERVAL_SEC = 0.6
RETRY = 3

NETWORK_CALLS = 0
_last_call = [0.0]


def _cache_path(path: str) -> Path:
    key = hashlib.sha1(path.encode('utf-8')).hexdigest()
    return CACHE_DIR / f'{key}.html'


def fetch(path: str) -> str:
    """path は '/timetable/...' 形式。UTF-8 デコード済み HTML を返す。"""
    global NETWORK_CALLS
    cache = _cache_path(path)
    if cache.exists():
        return cache.read_text(encoding='utf-8')

    CACHE_DIR.mkdir(parents=True, exist_ok=True)
    last_err = None
    for attempt in range(RETRY):
        wait = INTERVAL_SEC - (time.monotonic() - _last_call[0])
        if wait > 0:
            time.sleep(wait)
        try:
            res = requests.get(BASE + path, headers={'User-Agent': UA}, timeout=30)
            _last_call[0] = time.monotonic()
            NETWORK_CALLS += 1
            res.raise_for_status()
            html = res.content.decode('utf-8')
            cache.write_text(html, encoding='utf-8')
            return html
        except Exception as e:
            last_err = e
            _last_call[0] = time.monotonic()
            time.sleep(1.5 * (attempt + 1))
    raise RuntimeError(f'fetch failed after {RETRY} tries: {path}: {last_err}')
```

- [ ] **Step 5: テストを実行して通過を確認**

```bash
cd "$WORK" && python -m unittest discover -s tests -v
```

Expected: PASS（7テスト）

- [ ] **Step 6: フィクスチャを保存**

以降のパーサテストで使う実 HTML を固定する。

```bash
cd "$WORK" && python -c "
from pathlib import Path
import yahoo_fetch
Path('fixtures').mkdir(exist_ok=True)
pairs = [
    ('idx_hadano_up_wd.html',  '/timetable/23283/3090?kind=1'),
    ('idx_hadano_up_sa.html',  '/timetable/23283/3090?kind=2'),
    ('idx_hadano_up_ho.html',  '/timetable/23283/3090?kind=4'),
    ('detail_92748.html',      '/timetable/23283/3090/92748?kind=1&hh=8&mm=4'),
]
for name, path in pairs:
    Path('fixtures', name).write_text(yahoo_fetch.fetch(path), encoding='utf-8')
    print('saved', name)
"
```

Expected: 4ファイルが `WORK/fixtures/` に保存される

---

## Task 2: インデックスページのパーサ

**Files:**
- Create: `WORK/parse_index.py`
- Create: `WORK/tests/test_parse_index.py`

**Interfaces:**
- Consumes: `WORK/fixtures/idx_*.html`
- Produces:
  - `parse_index.parse(html: str) -> IndexPage`
  - `IndexPage` = `dataclass(trains: list[Train], type_legend: dict[str,str], dest_legend: dict[str,str], default_dest: str)`
  - `Train` = `dataclass(dep_min: int, type_name: str, dest: str, detail_path: str)`
    - `dep_min` は 0:00 からの経過分。0時台（`tr#hh_24`）は 1440 を加算済み
    - `type_name` は `'各駅停車'`〜の正式名。凡例から解決する
    - `dest` は行先の正式名。凡例の「無印」がデフォルト
    - `detail_path` は `/timetable/23283/3090/92748?kind=1&hh=8&mm=4` 形式

**HTML構造メモ（実ページで確認済み）:**

```html
<tr id="hh_8" class="even"><td class="hour">8</td><td><ul>
  <li class="timeNumb"><a href="/timetable/23283/3090/92748?kind=1&amp;hh=8&amp;mm=4">
    <dl style="color:#006600"><dt>4</dt><dd class="trainType">[快]</dd></dl></a></li>
  <li class="timeNumb"><a href="...&amp;mm=20">
    <dl><dt>20</dt><dd class="trainFor">町</dd></dl></a></li>
</ul></td></tr>
<table class="tblDiaNote"><tbody>
  <tr id="timeNotice1"><th>列車種別・列車名</th><td><ul>
    <li>無印<!-- -->：<!-- -->各駅停車</li><li>急<!-- -->：<!-- -->急行</li></ul></td></tr>
  <tr id="timeNotice2"><th>行き先・経由</th><td><ul>
    <li>無印<!-- -->：<!-- -->新宿</li><li>町<!-- -->：<!-- -->町田</li></ul></td></tr>
</tbody></table>
```

- `dd.trainType` が無い列車は「無印」＝各駅停車
- `dd.trainFor` が無い列車は「無印」＝方面の代表駅
- `tr#hh_24` の `td.hour` は `0` と表示されるが 0時台を指す
- `<!-- -->` はコメントノード。BeautifulSoup の `get_text()` で自然に消える
- 凡例はページごとに異なるので**ハードコードしない**

- [ ] **Step 1: 失敗するテストを書く**

`WORK/tests/test_parse_index.py`:

```python
import unittest
from pathlib import Path
import sys

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import parse_index

FIX = Path(__file__).resolve().parents[1] / 'fixtures'


class TestParseIndex(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.page = parse_index.parse((FIX / 'idx_hadano_up_wd.html').read_text(encoding='utf-8'))

    def test_legend_types(self):
        self.assertEqual(self.page.type_legend['無印'], '各駅停車')
        self.assertEqual(self.page.type_legend['急'], '急行')
        self.assertEqual(self.page.type_legend['快'], '快速急行')
        self.assertEqual(self.page.type_legend['特'], '特急')

    def test_legend_dests(self):
        self.assertEqual(self.page.default_dest, '新宿')
        self.assertEqual(self.page.dest_legend['町'], '町田')
        self.assertEqual(self.page.dest_legend['相'], '相模大野')
        self.assertEqual(self.page.dest_legend['厚'], '本厚木')

    def test_total_train_count(self):
        # 2026-07-31 時点の実ダイヤ。ダイヤ改正で変わるため範囲で確認する
        self.assertGreater(len(self.page.trains), 120)
        self.assertLess(len(self.page.trains), 200)

    def test_sorted_by_dep_min(self):
        mins = [t.dep_min for t in self.page.trains]
        self.assertEqual(mins, sorted(mins))

    def test_0804_is_kaisoku_kyuko_shinjuku(self):
        t = next(t for t in self.page.trains if t.dep_min == 8 * 60 + 4)
        self.assertEqual(t.type_name, '快速急行')
        self.assertEqual(t.dest, '新宿')
        self.assertIn('/92748', t.detail_path)

    def test_0820_is_local_to_machida(self):
        t = next(t for t in self.page.trains if t.dep_min == 8 * 60 + 20)
        self.assertEqual(t.type_name, '各駅停車')
        self.assertEqual(t.dest, '町田')

    def test_after_midnight_train_offset_by_1440(self):
        # 0:14発 町田行き → 1440 + 14
        t = next(t for t in self.page.trains if t.dep_min == 1440 + 14)
        self.assertEqual(t.dest, '町田')

    def test_no_train_between_1_and_2_am_boundary(self):
        # 1440 を超える列車が確実に末尾に来ている
        self.assertEqual(max(t.dep_min for t in self.page.trains), 1440 + 14)

    def test_detail_path_has_kind_and_time(self):
        t = self.page.trains[0]
        self.assertIn('kind=', t.detail_path)
        self.assertIn('hh=', t.detail_path)
        self.assertIn('mm=', t.detail_path)


class TestSaturdayHolidaySame(unittest.TestCase):
    def test_kind2_and_kind4_identical(self):
        sa = parse_index.parse((FIX / 'idx_hadano_up_sa.html').read_text(encoding='utf-8'))
        ho = parse_index.parse((FIX / 'idx_hadano_up_ho.html').read_text(encoding='utf-8'))
        key = lambda p: [(t.dep_min, t.type_name, t.dest) for t in p.trains]
        self.assertEqual(key(sa), key(ho), '土曜と日祝でダイヤが異なる。3区分格納に切り替える必要がある')


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 2: テストを実行して失敗を確認**

```bash
cd "$WORK" && python -m unittest tests.test_parse_index -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'parse_index'`

- [ ] **Step 3: `WORK/parse_index.py` を実装**

```python
"""時刻表インデックスページのパーサ。"""
import html as htmllib
import re
from dataclasses import dataclass, field

from bs4 import BeautifulSoup


@dataclass(frozen=True)
class Train:
    dep_min: int
    type_name: str
    dest: str
    detail_path: str


@dataclass
class IndexPage:
    trains: list = field(default_factory=list)
    type_legend: dict = field(default_factory=dict)
    dest_legend: dict = field(default_factory=dict)
    default_dest: str = ''


def _legend(soup, row_id):
    """凡例行 (#timeNotice1 等) を {記号: 正式名} に変換する。"""
    out = {}
    tr = soup.find('tr', id=row_id)
    if not tr:
        return out
    for li in tr.select('td li'):
        text = li.get_text().replace('\u3000', ' ').strip()
        m = re.match(r'^(.+?)\s*[：:]\s*(.+)$', text)
        if m:
            out[m.group(1).strip()] = m.group(2).strip()
    return out


def parse(html_text: str) -> IndexPage:
    soup = BeautifulSoup(html_text, 'html.parser')
    page = IndexPage()
    page.type_legend = _legend(soup, 'timeNotice1')
    page.dest_legend = _legend(soup, 'timeNotice2')
    page.default_dest = page.dest_legend.get('無印', '')
    default_type = page.type_legend.get('無印', '各駅停車')

    for tr in soup.select('tr[id^="hh_"]'):
        m = re.match(r'^hh_(\d+)$', tr.get('id', ''))
        if not m:
            continue
        hour = int(m.group(1))          # hh_24 は 0時台
        for li in tr.select('li.timeNumb'):
            a = li.find('a')
            dt = li.find('dt')
            if not a or not dt:
                continue
            minute = int(dt.get_text().strip())
            type_dd = li.find('dd', class_='trainType')
            dest_dd = li.find('dd', class_='trainFor')
            type_sym = type_dd.get_text().strip().strip('[]') if type_dd else '無印'
            dest_sym = dest_dd.get_text().strip() if dest_dd else '無印'
            page.trains.append(Train(
                dep_min=hour * 60 + minute,
                type_name=page.type_legend.get(type_sym, default_type),
                dest=page.dest_legend.get(dest_sym, page.default_dest),
                detail_path=htmllib.unescape(a.get('href', '')),
            ))

    page.trains.sort(key=lambda t: t.dep_min)
    return page
```

- [ ] **Step 4: テストを実行して通過を確認**

```bash
cd "$WORK" && python -m unittest tests.test_parse_index -v
```

Expected: PASS（11テスト）

`test_kind2_and_kind4_identical` が失敗した場合は、土曜と日祝でダイヤが違うということ。
その場合は Task 5 のデータ構造を `wd`/`sa`/`ho` の3区分に拡張し、
Task 8 の `diaKindOf()` も3区分を返すよう変更する。**この判断はユーザーに報告してから進める。**

---

## Task 3: 詳細ページのパーサ

**Files:**
- Create: `WORK/parse_detail.py`
- Create: `WORK/tests/test_parse_detail.py`

**Interfaces:**
- Consumes: `WORK/fixtures/detail_92748.html`
- Produces:
  - `parse_detail.parse(html_text: str) -> list[Stop]`
  - `Stop` = `dataclass(name: str, arr_min: int | None, dep_min: int | None)`
    - 分は 0:00 起点。ページ内で時刻が巻き戻ったら（例 23:58 → 0:05）以降に 1440 を加算する
  - `parse_detail.arrival_at(stops, station: str, dep_min_of_origin: int) -> int | None`
    - `station` が空文字なら**終着駅**（最後の停車駅）の着時刻を返す
    - 駅名は前方一致で照合する（`海老名(相鉄・小田急)` のような注記付きに対応）
    - 出発駅より前の停車駅はスキップする
    - 見つからなければ `None`

**HTML構造メモ（実ページで確認済み）:**

```html
<li><div class="station">
  <ul class="time"><li>8:36<!-- -->着</li><li>8:39<!-- -->発</li></ul>
  <p class="title">相模大野</p>
  <ul class="relLink">...</ul>
</div><p class="estimatedTime">2<!-- -->分</p></li>
```

- 始発駅は `発` のみ、終着駅は `着` のみ
- 駅名に `(相鉄・小田急)` のような注記が付くことがある

- [ ] **Step 1: 失敗するテストを書く**

`WORK/tests/test_parse_detail.py`:

```python
import unittest
from pathlib import Path
import sys

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import parse_detail

FIX = Path(__file__).resolve().parents[1] / 'fixtures'


class TestParseDetail(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.stops = parse_detail.parse((FIX / 'detail_92748.html').read_text(encoding='utf-8'))

    def test_has_stops(self):
        self.assertGreater(len(self.stops), 5)

    def test_first_stop_has_dep_only(self):
        self.assertIsNone(self.stops[0].arr_min)
        self.assertIsNotNone(self.stops[0].dep_min)

    def test_last_stop_has_arr_only(self):
        self.assertIsNotNone(self.stops[-1].arr_min)
        self.assertIsNone(self.stops[-1].dep_min)

    def test_sagamiono_arrival(self):
        # 実ページ: 相模大野 8:36着 8:39発
        s = next(s for s in self.stops if s.name.startswith('相模大野'))
        self.assertEqual(s.arr_min, 8 * 60 + 36)
        self.assertEqual(s.dep_min, 8 * 60 + 39)

    def test_ebina_name_with_suffix_is_kept(self):
        s = next(s for s in self.stops if s.name.startswith('海老名'))
        self.assertTrue(s.name.startswith('海老名'))

    def test_times_are_monotonic(self):
        seq = []
        for s in self.stops:
            if s.arr_min is not None:
                seq.append(s.arr_min)
            if s.dep_min is not None:
                seq.append(s.dep_min)
        self.assertEqual(seq, sorted(seq))


class TestArrivalAt(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.stops = parse_detail.parse((FIX / 'detail_92748.html').read_text(encoding='utf-8'))

    def test_named_station(self):
        got = parse_detail.arrival_at(self.stops, '相模大野', 8 * 60 + 4)
        self.assertEqual(got, 8 * 60 + 36)

    def test_terminal_when_station_is_empty(self):
        got = parse_detail.arrival_at(self.stops, '', 8 * 60 + 4)
        self.assertEqual(got, self.stops[-1].arr_min)

    def test_prefix_match(self):
        got = parse_detail.arrival_at(self.stops, '海老名', 8 * 60 + 4)
        self.assertIsNotNone(got)

    def test_unknown_station_returns_none(self):
        self.assertIsNone(parse_detail.arrival_at(self.stops, '存在しない駅', 8 * 60 + 4))

    def test_station_before_origin_is_skipped(self):
        # 秦野 8:04発の列車。出発より前の駅を指定しても拾わない
        got = parse_detail.arrival_at(self.stops, '新松田', 8 * 60 + 4)
        self.assertIsNone(got)


class TestMidnightRollover(unittest.TestCase):
    def test_rollover_adds_1440(self):
        html_text = '''
        <ul><li><div class="station"><ul class="time"><li>23:58<!-- -->発</li></ul>
        <p class="title">A駅</p></div></li>
        <li><div class="station"><ul class="time"><li>0:05<!-- -->着</li><li>0:06<!-- -->発</li></ul>
        <p class="title">B駅</p></div></li>
        <li><div class="station"><ul class="time"><li>0:20<!-- -->着</li></ul>
        <p class="title">C駅</p></div></li></ul>
        '''
        stops = parse_detail.parse(html_text)
        self.assertEqual(stops[0].dep_min, 23 * 60 + 58)
        self.assertEqual(stops[1].arr_min, 1440 + 5)
        self.assertEqual(stops[2].arr_min, 1440 + 20)


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 2: テストを実行して失敗を確認**

```bash
cd "$WORK" && python -m unittest tests.test_parse_detail -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'parse_detail'`

- [ ] **Step 3: `WORK/parse_detail.py` を実装**

```python
"""列車詳細ページのパーサ。"""
import re
from dataclasses import dataclass

from bs4 import BeautifulSoup


@dataclass
class Stop:
    name: str
    arr_min: int = None
    dep_min: int = None


def _to_min(text):
    m = re.match(r'^(\d{1,2}):(\d{2})$', text.strip())
    return int(m.group(1)) * 60 + int(m.group(2)) if m else None


def parse(html_text: str):
    soup = BeautifulSoup(html_text, 'html.parser')
    stops = []
    for div in soup.select('div.station'):
        title = div.find('p', class_='title')
        if not title:
            continue
        stop = Stop(name=title.get_text().strip())
        for li in div.select('ul.time li'):
            text = li.get_text().strip()
            value = _to_min(text.rstrip('着発'))
            if value is None:
                continue
            if text.endswith('着'):
                stop.arr_min = value
            elif text.endswith('発'):
                stop.dep_min = value
        if stop.arr_min is not None or stop.dep_min is not None:
            stops.append(stop)

    # 日跨ぎ補正: 時刻が巻き戻ったら以降に 1440 を加算する
    offset, prev = 0, -1
    for s in stops:
        for attr in ('arr_min', 'dep_min'):
            v = getattr(s, attr)
            if v is None:
                continue
            if v + offset < prev:
                offset += 1440
            v += offset
            setattr(s, attr, v)
            prev = v
    return stops


def arrival_at(stops, station: str, dep_min_of_origin: int):
    """station への到着分を返す。station が空なら終着駅。見つからなければ None。"""
    if not stops:
        return None
    if station == '':
        return stops[-1].arr_min
    for s in stops:
        ref = s.arr_min if s.arr_min is not None else s.dep_min
        if ref is None or ref < dep_min_of_origin:
            continue
        if s.name.startswith(station):
            return s.arr_min if s.arr_min is not None else s.dep_min
    return None
```

- [ ] **Step 4: テストを実行して通過を確認**

```bash
cd "$WORK" && python -m unittest tests.test_parse_detail -v
```

Expected: PASS（12テスト）

---

## Task 4: 固定所要時間の実測検証

短区間4ボード（G2/G3/K1/K2）に使う所要時間を、実データで検証してから確定する。

**Files:**
- Create: `WORK/calibrate.py`
- Create: `WORK/out/calibration.md`

**Interfaces:**
- Consumes: `boards.BOARDS`、`yahoo_fetch.fetch`、`parse_index.parse`、`parse_detail.parse` / `arrival_at`
- Produces: `WORK/out/fixed_ride.json` — `{board_id: {type_name: minutes}}`

- [ ] **Step 1: `WORK/calibrate.py` を作成**

```python
"""短区間ボードの所要時間を実測し、種別ごとに一定かを検証する。"""
import json
from collections import defaultdict
from pathlib import Path

import boards
import parse_detail
import parse_index
import yahoo_fetch

SAMPLES_PER_TYPE = 5
OUT = Path(__file__).parent / 'out'


def main():
    OUT.mkdir(exist_ok=True)
    fixed = {}
    report = ['# 固定所要時間 実測結果', '']

    for b in boards.BOARDS:
        if b.mode != 'fixed':
            continue
        page = parse_index.parse(yahoo_fetch.fetch(f'/timetable/{b.station_id}/{b.line_id}?kind=1'))
        by_type = defaultdict(list)
        for t in page.trains:
            by_type[t.type_name].append(t)

        report.append(f'## {b.id} {b.station} → {b.arr_station}')
        board_fixed, unstable = {}, []
        for type_name, trains in sorted(by_type.items()):
            step = max(1, len(trains) // SAMPLES_PER_TYPE)
            picked = trains[::step][:SAMPLES_PER_TYPE]
            rides = []
            for t in picked:
                stops = parse_detail.parse(yahoo_fetch.fetch(t.detail_path))
                arr = parse_detail.arrival_at(stops, b.arr_station, t.dep_min)
                if arr is not None:
                    rides.append(arr - t.dep_min)
            if not rides:
                report.append(f'- {type_name}: 取得できず')
                unstable.append(type_name)
                continue
            uniq = sorted(set(rides))
            report.append(f'- {type_name}: {rides} → 候補 {uniq}')
            if len(uniq) == 1:
                board_fixed[type_name] = uniq[0]
            else:
                unstable.append(type_name)

        if unstable:
            report.append(f'- **ばらつきあり: {unstable} → このボードは詳細実取得へ切り替えを検討**')
        fixed[b.id] = board_fixed
        report.append('')

    (OUT / 'fixed_ride.json').write_text(
        json.dumps(fixed, ensure_ascii=False, indent=2), encoding='utf-8')
    (OUT / 'calibration.md').write_text('\n'.join(report), encoding='utf-8')
    print('\n'.join(report))


if __name__ == '__main__':
    main()
```

- [ ] **Step 2: 実行して結果を確認**

```bash
cd "$WORK" && python calibrate.py
```

Expected: 4ボード分の所要時間が出力される。取得は約80リクエスト（約1分）

- [ ] **Step 3: 結果を判定してユーザーに報告**

`WORK/out/calibration.md` を読み、以下を報告する。

- 全ボード・全種別で所要時間が一意 → そのまま Task 5 へ進む
- ばらつきのあるボードがある → **そのボードを `boards.py` で `mode='detail'` に変更する**。
  追加取得量（そのボードの列車数 × 2区分）を添えてユーザーに報告してから変更する

---

## Task 5: 全データ取得とJS生成

**Files:**
- Create: `WORK/holidays.py`
- Create: `WORK/build.py`
- Create: `WORK/verify.py`
- Create: `WORK/tests/test_build.py`
- Create: `WORK/out/timetable.js`
- Create: `WORK/out/verify.md`

**Interfaces:**
- Consumes: Task 1〜4 の全モジュール、`WORK/out/fixed_ride.json`
- Produces:
  - `build.build() -> dict` — `{board_id: {'wd': [[dep,typeIdx,destIdx,arr], ...], 'ho': [...]}}`
  - `build.emit_js(data, types, dests, holidays) -> str` — `index.html` に貼る JS テキスト
  - `holidays.list_dates(2026, 2027) -> list[str]` — `'YYYY-MM-DD'` の昇順リスト
  - `WORK/out/timetable.js` — 生成物

- [ ] **Step 1: `WORK/holidays.py` を作成**

```python
"""祝日＋年末年始のリスト。鉄道各社は年末年始を土休日ダイヤで運行する。"""
import datetime

import jpholiday


def list_dates(start_year: int, end_year: int):
    out = set()
    for year in range(start_year, end_year + 1):
        for d, _ in jpholiday.year_holidays(year):
            out.add(d.isoformat())
        for md in [(12, 29), (12, 30), (12, 31), (1, 1), (1, 2), (1, 3)]:
            out.add(datetime.date(year, md[0], md[1]).isoformat())
    return sorted(out)
```

- [ ] **Step 2: 失敗するテストを書く**

`WORK/tests/test_build.py`:

```python
import unittest
from pathlib import Path
import sys

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import build
import holidays


class TestHolidays(unittest.TestCase):
    def test_includes_ganjitsu(self):
        self.assertIn('2026-01-01', holidays.list_dates(2026, 2027))

    def test_includes_year_end(self):
        self.assertIn('2026-12-30', holidays.list_dates(2026, 2027))

    def test_sorted_and_unique(self):
        d = holidays.list_dates(2026, 2027)
        self.assertEqual(d, sorted(set(d)))

    def test_covers_two_years(self):
        d = holidays.list_dates(2026, 2027)
        self.assertTrue(any(x.startswith('2026-') for x in d))
        self.assertTrue(any(x.startswith('2027-') for x in d))


class TestEmitJs(unittest.TestCase):
    def test_emit_shape(self):
        data = {'G1': {'wd': [[484, 3, 0, 512]], 'ho': [[500, 0, 1, -1]]}}
        js = build.emit_js(data, ['各停', '準急', '急行', '快急'], ['新宿', '町田'], ['2026-01-01'])
        self.assertIn('const TT_TYPES', js)
        self.assertIn('const TT_DESTS', js)
        self.assertIn('const TT_HOLIDAYS', js)
        self.assertIn('const TIMETABLE', js)
        self.assertIn('[484,3,0,512]', js.replace(' ', ''))

    def test_emit_is_ascii_safe_for_numbers(self):
        data = {'G1': {'wd': [[0, 0, 0, -1]], 'ho': []}}
        js = build.emit_js(data, ['各停'], ['新宿'], [])
        self.assertNotIn('None', js)
        self.assertNotIn('True', js)


class TestRowInvariants(unittest.TestCase):
    """生成済みデータがあれば全件検査する。無ければスキップ。"""

    @classmethod
    def setUpClass(cls):
        cls.path = Path(__file__).resolve().parents[1] / 'out' / 'data.json'
        if not cls.path.exists():
            raise unittest.SkipTest('build.py 未実行')
        import json
        cls.data = json.loads(cls.path.read_text(encoding='utf-8'))

    def test_arrival_after_departure(self):
        for bid, kinds in self.data.items():
            for kind, rows in kinds.items():
                for row in rows:
                    if row[3] >= 0:
                        self.assertGreater(row[3], row[0], f'{bid}/{kind}/{row}')

    def test_departures_sorted(self):
        for bid, kinds in self.data.items():
            for kind, rows in kinds.items():
                mins = [r[0] for r in rows]
                self.assertEqual(mins, sorted(mins), f'{bid}/{kind}')

    def test_all_eight_boards_present(self):
        self.assertEqual(sorted(self.data.keys()),
                         ['G1', 'G2', 'G3', 'G4', 'K1', 'K2', 'K3', 'K4'])

    def test_each_board_has_trains(self):
        for bid, kinds in self.data.items():
            for kind, rows in kinds.items():
                self.assertGreater(len(rows), 50, f'{bid}/{kind} の本数が少なすぎる')


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 3: テストを実行して失敗を確認**

```bash
cd "$WORK" && python -m unittest tests.test_build -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'build'`

- [ ] **Step 4: `WORK/build.py` を実装**

```python
"""全ボードのデータを組み立てて JS を出力する。"""
import json
from pathlib import Path

import boards
import holidays
import parse_detail
import parse_index
import yahoo_fetch

OUT = Path(__file__).parent / 'out'
# 表示用の短縮種別名。凡例の正式名から引く
TYPE_SHORT = {
    '各駅停車': '各停', '普通': '各停', '準急': '準急', '区間準急': '区急',
    '急行': '急行', '通勤急行': '通急', '快速急行': '快急', '通勤快急': '通快',
    '特急': '特急', '特急ロマンスカー': '特急', '準特急': '準特',
}


def short_type(name):
    return TYPE_SHORT.get(name, name[:2])


def build():
    fixed = json.loads((OUT / 'fixed_ride.json').read_text(encoding='utf-8'))
    types, dests = [], []
    data = {}

    def idx(pool, value):
        if value not in pool:
            pool.append(value)
        return pool.index(value)

    for b in boards.BOARDS:
        data[b.id] = {}
        for kind_key, kind_no in (('wd', 1), ('ho', 4)):
            page = parse_index.parse(
                yahoo_fetch.fetch(f'/timetable/{b.station_id}/{b.line_id}?kind={kind_no}'))
            rows = []
            for t in page.trains:
                if b.mode == 'detail':
                    stops = parse_detail.parse(yahoo_fetch.fetch(t.detail_path))
                    arr = parse_detail.arrival_at(stops, b.arr_station, t.dep_min)
                else:
                    ride = fixed.get(b.id, {}).get(t.type_name)
                    arr = t.dep_min + ride if ride is not None else None
                rows.append([
                    t.dep_min,
                    idx(types, short_type(t.type_name)),
                    idx(dests, t.dest),
                    arr if arr is not None else -1,
                ])
            data[b.id][kind_key] = rows
            print(f'{b.id}/{kind_key}: {len(rows)}本')

    OUT.mkdir(exist_ok=True)
    (OUT / 'data.json').write_text(json.dumps(data, ensure_ascii=False), encoding='utf-8')
    return data, types, dests


def emit_js(data, types, dests, holiday_list):
    lines = []
    lines.append('// ---- 時刻表データ（自動生成。再取得時は tw_build/build.py を実行）----')
    lines.append(f'const TT_TYPES = {json.dumps(types, ensure_ascii=False)};')
    lines.append(f'const TT_DESTS = {json.dumps(dests, ensure_ascii=False)};')
    lines.append(f'const TT_HOLIDAYS = new Set({json.dumps(holiday_list)});')
    lines.append('const TIMETABLE = {')
    for b in boards.BOARDS:
        d = data.get(b.id, {'wd': [], 'ho': []})
        lines.append(f"  {b.id}: {{")
        lines.append(f"    line: {json.dumps(b.line, ensure_ascii=False)},")
        lines.append(f"    arrLabel: {json.dumps(b.arr_label, ensure_ascii=False)},")
        for kind_key in ('wd', 'ho'):
            rows = ','.join('[' + ','.join(str(v) for v in r) + ']' for r in d[kind_key])
            lines.append(f'    {kind_key}: [{rows}],')
        lines.append('  },')
    lines.append('};')
    return '\n'.join(lines)


def main():
    data, types, dests = build()
    js = emit_js(data, types, dests, holidays.list_dates(2026, 2027))
    (OUT / 'timetable.js').write_text(js, encoding='utf-8')
    print(f'wrote timetable.js: {len(js)} bytes')


if __name__ == '__main__':
    main()
```

- [ ] **Step 5: 取得を実行**

```bash
cd "$WORK" && python build.py
```

Expected: 8ボード × 2区分の本数が出力され、`out/timetable.js` が生成される。
初回は約1,480リクエストで15〜20分かかる。キャッシュがあるため再実行は数秒。

進捗が見えないため、**バックグラウンド実行にしてログを監視する**こと。

- [ ] **Step 6: テストを実行して通過を確認**

```bash
cd "$WORK" && python -m unittest discover -s tests -v
```

Expected: PASS（全テスト。`TestRowInvariants` の4テストも実データで通ること）

- [ ] **Step 7: `WORK/verify.py` を作成して整合性レポートを出す**

```python
"""生成データを Yahoo の表示と突き合わせるためのレポートを出す。"""
import json
from pathlib import Path

import boards

OUT = Path(__file__).parent / 'out'


def fmt(m):
    if m < 0:
        return '--:--'
    return f'{(m // 60) % 24:02d}:{m % 60:02d}'


def main():
    data = json.loads((OUT / 'data.json').read_text(encoding='utf-8'))
    types = json.loads((OUT / 'timetable.js').read_text(encoding='utf-8')
                       .split('const TT_TYPES = ')[1].split(';')[0])
    lines = ['# 生成データ検証レポート', '']
    total_missing = 0
    for b in boards.BOARDS:
        for kind in ('wd', 'ho'):
            rows = data[b.id][kind]
            missing = sum(1 for r in rows if r[3] < 0)
            total_missing += missing
            lines.append(f'## {b.id} {b.station} ({b.line}) / {kind}')
            lines.append(f'- 本数: {len(rows)}')
            lines.append(f'- 始発: {fmt(rows[0][0])} / 終電: {fmt(rows[-1][0])}')
            lines.append(f'- 到着時刻なし: {missing}件')
            sample = rows[:3]
            for r in sample:
                lines.append(f'  - {fmt(r[0])} {types[r[1]]} → {b.arr_label} {fmt(r[3])}')
            lines.append('')
    lines.insert(1, f'到着時刻なしの合計: {total_missing}件')
    (OUT / 'verify.md').write_text('\n'.join(lines), encoding='utf-8')
    print('\n'.join(lines))


if __name__ == '__main__':
    main()
```

- [ ] **Step 8: 検証レポートを確認してユーザーに報告**

```bash
cd "$WORK" && python verify.py
```

各ボードの本数・始発・終電を Yahoo のページ表示と突き合わせる。
到着時刻なしの件数が全体の1%を超える場合はパーサの不具合を疑い、原因を調べてから次へ進む。

---

## Task 6: index.html に CSS と DOM を追加

この時点では追跡パネルは常に非表示。**既存の見た目が1ピクセルも変わらないことを確認する**のがゴール。

**Files:**
- Modify: `PROJ/index.html`（`<style>` 末尾、`#weather-links` の直後）

**Interfaces:**
- Consumes: なし
- Produces: DOM 要素
  - `#tracker-wrap`（全体ラッパー。既定 `display:none`）
  - `#tracker-toggle`（追跡 ON/OFF ボタン）
  - `#tracker-body`（方向・駅・リストのまとまり）
  - `#tracker-dir`（方向トグルの入れ物）
  - `#tracker-stations`（駅ボタンの入れ物）
  - `#tracker-list`（スクロール領域）

- [ ] **Step 1: 改修前のバイト情報を記録**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
print('bytes',len(b),'CRLF',b.count(b'\r\n'),'LF',b.count(b'\n'),'BOM',b[:3]==b'\xef\xbb\xbf')
"
```

Expected: `bytes 42639 CRLF 1128 LF 1128 BOM False`

- [ ] **Step 2: `<style>` の末尾（`@keyframes pulse { ... }` の直後）に CSS を追加**

```css
    /* 追跡パネル */
    #tracker-wrap {
      padding: 0 16px 8px;
      display: none;
    }
    #tracker-toggle {
      background: none;
      border: 1px solid #2d2d2d;
      border-radius: 6px;
      color: #666;
      font-size: 0.72rem;
      font-weight: 600;
      padding: 4px 12px;
      cursor: pointer;
      font-family: inherit;
    }
    #tracker-toggle.on {
      border-color: #26C6DA;
      color: #26C6DA;
    }
    #tracker-body { display: none; margin-top: 8px; }
    #tracker-body.on { display: block; }
    #tracker-dir {
      display: flex;
      gap: 18px;
      font-size: 0.95rem;
      font-weight: 700;
      margin-bottom: 6px;
    }
    .tracker-dir-btn {
      background: none;
      border: none;
      color: #555;
      cursor: pointer;
      padding: 2px 0;
      font-size: inherit;
      font-weight: inherit;
      font-family: inherit;
    }
    .tracker-dir-btn.sel { color: #CE93D8; }
    #tracker-stations {
      display: flex;
      flex-wrap: wrap;
      gap: 6px;
      margin-bottom: 6px;
    }
    .tracker-st-btn {
      background: #1e1e1e;
      border: 1px solid #2d2d2d;
      border-radius: 6px;
      color: #9e9e9e;
      font-size: 0.8rem;
      padding: 4px 10px;
      cursor: pointer;
      font-family: inherit;
      white-space: nowrap;
    }
    .tracker-st-btn.sel {
      border-color: #26C6DA;
      color: #26C6DA;
      font-weight: 700;
    }
    #tracker-list {
      background: #1e1e1e;
      border-radius: 10px;
      padding: 6px 8px;
      max-height: 260px;
      overflow-y: auto;
      font-size: 0.82rem;
    }
    .tracker-row {
      display: flex;
      align-items: center;
      line-height: 1.9em;
      white-space: nowrap;
    }
    .tracker-row.past { opacity: 0.4; }
    .tracker-row.next { font-weight: 700; }
    .tr-mark { width: 1em; flex-shrink: 0; color: #26C6DA; }
    .tr-dep  { width: 3.4em; flex-shrink: 0; }
    .tr-bang { width: 0.8em; flex-shrink: 0; color: #EA4335; font-weight: 700; }
    .tr-type { width: 2.6em; flex-shrink: 0; }
    .tr-dest { width: 4.5em; flex-shrink: 0; overflow: hidden; text-overflow: ellipsis; }
    .tr-arr  { flex: 1; color: #9e9e9e; overflow: hidden; text-overflow: ellipsis; }
    .tr-left { width: 3.6em; flex-shrink: 0; text-align: right; }
    #tracker-empty { color: #666; font-size: 0.8rem; padding: 6px 0; }
```

- [ ] **Step 3: `#weather-links` の閉じ `</div>` の直後に DOM を追加**

`index.html:265` の `</div>` と `<div id="cards">` の間に挿入する。

```html
<div id="tracker-wrap">
  <button id="tracker-toggle" onclick="toggleTracker()">追跡 OFF</button>
  <div id="tracker-body">
    <div id="tracker-dir"></div>
    <div id="tracker-stations"></div>
    <div id="tracker-list"></div>
  </div>
</div>
```

- [ ] **Step 4: エンコーディングと改行を検証**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
crlf, lf = b.count(b'\r\n'), b.count(b'\n')
print('bytes',len(b),'CRLF',crlf,'LF',lf,'BOM',b[:3]==b'\xef\xbb\xbf')
assert crlf == lf, 'LF単独の行が混入している'
assert b[:3] != b'\xef\xbb\xbf', 'BOMが付いた'
b.decode('utf-8')
print('OK: UTF-8 / CRLF only / no BOM')
"
```

Expected: `OK: UTF-8 / CRLF only / no BOM`

- [ ] **Step 5: 既存表示が変わらないことを確認**

`index.html` をブラウザで開き、以下を目視確認する。

- 天気モード・CDモード・ALLモードの3モードを一巡し、表示が改修前と同じ
- 追跡パネルはどのモードでも表示されない（`#tracker-wrap` が `display:none` のまま）
- ブラウザのコンソールにエラーが出ていない

- [ ] **Step 6: 差分を確認してコミットコマンドを提示**

```bash
cd "$PROJ" && git diff --stat
```

ユーザーに以下を提示する（**実行しない**）。

```bash
cd "C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch"
git add index.html SPEC_tracker.md
git commit -m "追跡パネルのCSSとDOMを追加（非表示状態）"
```

---

## Task 7: 追跡パネルの状態管理

ON/OFF・方向・駅の切替と localStorage 復元まで。リストはまだ空。

**Files:**
- Modify: `PROJ/index.html`（`<script>` 内、カウントダウンのセクションと「起動」セクションの間）

**Interfaces:**
- Consumes: `TIMETABLE`（Task 10 で埋め込むまでは空オブジェクトで代用）
- Produces:
  - `TRACKER_BOARDS` — `{go: [{id, station}], back: [...]}` 経路順の配列
  - `trackerOn: boolean` / `trackerDir: 'go'|'back'` / `trackerBoardId: string`
  - `toggleTracker()` — ON/OFF 切替＋localStorage 保存
  - `setTrackerDir(dir)` — 方向切替＋既定駅へリセット
  - `setTrackerBoard(id)` — 駅切替
  - `renderTrackerControls()` — 方向・駅ボタンの再描画
  - `applyTrackerVisibility()` — `displayMode` に応じたラッパー表示制御
  - `defaultTrackerDir(now)` — `2:00〜11:59 → 'go'`、それ以外 `'back'`
  - `LS_TRACKER_ON` — `'trainwatch_tracker_on'`

- [ ] **Step 1: JS を追加**

`index.html` の `// 起動` セクション（`updateClock();` の行）の**直前**に挿入する。

```javascript
// ============================================================
// 追跡パネル
// ============================================================
const LS_TRACKER_ON = 'trainwatch_tracker_on';

// 経路順に並べたボード。ID は TIMETABLE のキーと一致
const TRACKER_BOARDS = {
  go:   [{ id:'G1', station:'秦野' },   { id:'G2', station:'相模大野' },
         { id:'G3', station:'中央林間' }, { id:'G4', station:'南町田' }],
  back: [{ id:'K1', station:'南町田' }, { id:'K2', station:'中央林間' },
         { id:'K3', station:'相模大野' }, { id:'K4', station:'秦野' }],
};
const TRACKER_DEFAULT_BOARD = { go: 'G1', back: 'K2' };

let trackerOn      = false;
let trackerDir     = 'go';
let trackerBoardId = 'G1';

// 2:00〜11:59 は行き、それ以外は帰り
function defaultTrackerDir(now) {
  const h = now.getHours();
  return (h >= 2 && h < 12) ? 'go' : 'back';
}

function applyTrackerVisibility() {
  const wrap = document.getElementById('tracker-wrap');
  wrap.style.display = (displayMode === 'both') ? 'block' : 'none';
}

function toggleTracker() {
  trackerOn = !trackerOn;
  try { localStorage.setItem(LS_TRACKER_ON, trackerOn ? '1' : '0'); } catch(e) {}
  renderTrackerControls();
  renderTrackerList();
}

function setTrackerDir(dir) {
  trackerDir     = dir;
  trackerBoardId = TRACKER_DEFAULT_BOARD[dir];
  renderTrackerControls();
  renderTrackerList();
}

function setTrackerBoard(id) {
  trackerBoardId = id;
  renderTrackerControls();
  renderTrackerList();
}

function renderTrackerControls() {
  const btn = document.getElementById('tracker-toggle');
  btn.textContent = trackerOn ? '追跡 ON' : '追跡 OFF';
  btn.className   = trackerOn ? 'on' : '';
  document.getElementById('tracker-body').className = trackerOn ? 'on' : '';
  if (!trackerOn) return;

  const dirEl = document.getElementById('tracker-dir');
  dirEl.innerHTML =
    `<button class="tracker-dir-btn${trackerDir==='go'?' sel':''}" ` +
    `onclick="setTrackerDir('go')">△行き</button>` +
    `<button class="tracker-dir-btn${trackerDir==='back'?' sel':''}" ` +
    `onclick="setTrackerDir('back')">▼帰り</button>`;

  document.getElementById('tracker-stations').innerHTML =
    TRACKER_BOARDS[trackerDir].map(b =>
      `<button class="tracker-st-btn${b.id===trackerBoardId?' sel':''}" ` +
      `onclick="setTrackerBoard('${b.id}')">${escHtml(b.station)}</button>`
    ).join('');
}

function initTracker() {
  try { trackerOn = localStorage.getItem(LS_TRACKER_ON) === '1'; } catch(e) { trackerOn = false; }
  trackerDir     = defaultTrackerDir(new Date());
  trackerBoardId = TRACKER_DEFAULT_BOARD[trackerDir];
  renderTrackerControls();
  renderTrackerList();
}
```

- [ ] **Step 2: `renderTrackerList()` の仮実装を追加**

Task 8 で本実装に差し替える。上記ブロックの末尾に置く。

```javascript
function renderTrackerList() {
  document.getElementById('tracker-list').innerHTML =
    '<div id="tracker-empty">（時刻表データ未投入）</div>';
}
```

- [ ] **Step 3: `applyDisplayMode()` に1行追加**

`index.html` の `applyDisplayMode()` 内、`refreshCountdownButton();` の**直前**に追加する。

```javascript
  applyTrackerVisibility();
```

追加後の該当箇所は次の形になる。

```javascript
  if (showCountdown) {
    refillFromSchedule();
    updateCountdown();
  } else {
    countdownTargets = [];
  }
  applyTrackerVisibility();
  refreshCountdownButton();
}
```

- [ ] **Step 4: 起動処理に `initTracker()` を追加**

`applyDisplayMode(initialMode);` の**直後**に1行追加する。

```javascript
initTracker();
```

- [ ] **Step 5: エンコーディングを検証**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
assert b.count(b'\r\n') == b.count(b'\n'), 'LF単独の行が混入'
assert b[:3] != b'\xef\xbb\xbf', 'BOM混入'
b.decode('utf-8')
print('OK', len(b), 'bytes')
"
```

Expected: `OK` と出る

- [ ] **Step 6: ブラウザで手動確認**

- ALL モードにすると `追跡 OFF` ボタンが出る。天気モード・CD モードでは出ない
- ボタンを押すと `追跡 ON` に変わり、方向トグルと駅ボタンが出る
- 朝（2:00〜11:59）に開くと `△行き`＋`秦野` が選択済み。昼以降は `▼帰り`＋`中央林間`
- リロードしても ON/OFF が保持される
- 方向を切り替えると駅の並びが入れ替わり、既定駅が選ばれる
- コンソールにエラーが出ない
- 天気・カウントダウン・電車カードの表示が変わっていない

- [ ] **Step 7: コミットコマンドを提示（実行しない）**

```bash
cd "C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch"
git add index.html
git commit -m "追跡パネルの状態管理を追加（ON/OFF・方向・駅）"
```

---

## Task 8: 列車選択ロジックと描画

**Files:**
- Modify: `PROJ/index.html`（`renderTrackerList()` の仮実装を差し替え）
- Create: `PROJ/_selftest.html`（検証用。Task 10 で削除）

**Interfaces:**
- Consumes: `TIMETABLE` / `TT_TYPES` / `TT_DESTS` / `TT_HOLIDAYS`、`trackerBoardId`、`colorOf()`（既存）
- Produces:
  - `trackerNowMin(now) -> number` — 0:00〜1:59 は `+1440`
  - `trackerDiaKind(now) -> 'wd'|'ho'` — 土日・祝日リスト該当日は `'ho'`。0:00〜1:59 は前日で判定
  - `pickTrains(rows, nowMin) -> {list: Array, nextIdx: number}` — 直前1本＋次9本
  - `fmtHHMM(min) -> string` — `1454` → `'00:14'`
  - `renderTrackerList(keepScroll)` — 本実装。`keepScroll` が真ならスクロール位置を維持する。
    1秒ごとの再描画は `renderTrackerList(true)` で呼ぶ。Task 7 の引数なし呼び出しは
    ボード切替時なのでスクロール位置を先頭に戻す（＝引数を足す必要はない）

- [ ] **Step 1: `_selftest.html` に失敗するテストを書く**

`PROJ/_selftest.html` を新規作成する。

```html
<!DOCTYPE html>
<html lang="ja"><head><meta charset="UTF-8"><title>tracker selftest</title>
<style>body{font-family:monospace;background:#111;color:#ddd;padding:16px}
.p{color:#4caf50}.f{color:#f44336;font-weight:700}</style></head><body>
<div id="out"></div>
<script>
// index.html から検証対象の純粋関数だけをコピーする。
// 実装を変えたらこちらも同じ内容に更新すること。
const TT_HOLIDAYS = new Set(['2026-01-01','2026-05-05']);

function trackerNowMin(now) {
  const m = now.getHours() * 60 + now.getMinutes();
  return m < 120 ? m + 1440 : m;
}

function trackerDiaKind(now) {
  const d = new Date(now);
  if (d.getHours() < 2) d.setDate(d.getDate() - 1);
  const pad = n => String(n).padStart(2, '0');
  const key = `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}`;
  if (TT_HOLIDAYS.has(key)) return 'ho';
  const w = d.getDay();
  return (w === 0 || w === 6) ? 'ho' : 'wd';
}

function pickTrains(rows, nowMin) {
  let nextIdx = rows.findIndex(r => r[0] >= nowMin);
  if (nextIdx < 0) nextIdx = rows.length;
  const start = Math.max(0, nextIdx - 1);
  const list  = rows.slice(start, start + 10);
  return { list, nextIdx: nextIdx - start };
}

function fmtHHMM(min) {
  const pad = n => String(n).padStart(2, '0');
  return `${pad(Math.floor(min / 60) % 24)}:${pad(min % 60)}`;
}

// ---- テスト ----
const results = [];
function eq(name, got, want) {
  const ok = JSON.stringify(got) === JSON.stringify(want);
  results.push({ name, ok, got, want });
}

eq('nowMin 昼', trackerNowMin(new Date(2026,6,31,8,13)), 493);
eq('nowMin 0時台は+1440', trackerNowMin(new Date(2026,6,31,0,14)), 1454);
eq('nowMin 1:59は+1440', trackerNowMin(new Date(2026,6,31,1,59)), 1559);
eq('nowMin 2:00はそのまま', trackerNowMin(new Date(2026,6,31,2,0)), 120);

eq('平日金曜', trackerDiaKind(new Date(2026,6,31,8,0)), 'wd');
eq('土曜', trackerDiaKind(new Date(2026,7,1,8,0)), 'ho');
eq('日曜', trackerDiaKind(new Date(2026,7,2,8,0)), 'ho');
eq('祝日リスト該当', trackerDiaKind(new Date(2026,4,5,8,0)), 'ho');
eq('土曜0:30は前日金曜=wd', trackerDiaKind(new Date(2026,7,1,0,30)), 'wd');
eq('日曜0:30は前日土曜=ho', trackerDiaKind(new Date(2026,7,2,0,30)), 'ho');

const rows = [[480,0,0,500],[490,1,0,505],[500,0,0,520],[510,1,0,530],
              [520,0,0,540],[530,1,0,550],[540,0,0,560],[550,1,0,570],
              [560,0,0,580],[570,1,0,590],[580,0,0,600],[590,1,0,610]];
let r = pickTrains(rows, 495);
eq('直前1本を含む', r.list[0][0], 490);
eq('次発の位置', r.nextIdx, 1);
eq('10件', r.list.length, 10);
r = pickTrains(rows, 470);
eq('始発前は先頭から', r.list[0][0], 480);
eq('始発前の次発位置', r.nextIdx, 0);
r = pickTrains(rows, 600);
eq('終電後は末尾のみ', r.list.length, 1);
eq('終電後の次発位置は範囲外', r.nextIdx, 1);
r = pickTrains(rows, 590);
eq('最終列車が次発', r.list[r.nextIdx][0], 590);

eq('fmt 通常', fmtHHMM(493), '08:13');
eq('fmt 0時台', fmtHHMM(1454), '00:14');
eq('fmt 24時ちょうど', fmtHHMM(1440), '00:00');

const pass = results.filter(x => x.ok).length;
document.getElementById('out').innerHTML =
  `<h2>${pass} / ${results.length} PASS</h2>` +
  results.map(x => x.ok
    ? `<div class="p">PASS ${x.name}</div>`
    : `<div class="f">FAIL ${x.name}: got ${JSON.stringify(x.got)} want ${JSON.stringify(x.want)}</div>`
  ).join('');
</script></body></html>
```

- [ ] **Step 2: `_selftest.html` をブラウザで開いて全パスを確認**

Expected: `21 / 21 PASS`

パスしない場合は、`_selftest.html` 内の関数実装を直してから Step 3 へ進む。
**`_selftest.html` で通った関数をそのまま `index.html` にコピーする**のが手順の要点。

- [ ] **Step 3: `index.html` の `renderTrackerList()` 仮実装を本実装に差し替える**

Task 7 で入れた仮の `renderTrackerList()` を削除し、以下に置き換える。
`trackerNowMin` / `trackerDiaKind` / `pickTrains` / `fmtHHMM` は
`_selftest.html` で検証したものと**同一の内容**にする。

```javascript
function trackerNowMin(now) {
  const m = now.getHours() * 60 + now.getMinutes();
  return m < 120 ? m + 1440 : m;
}

// 0:00〜1:59 は前日のダイヤを使う
function trackerDiaKind(now) {
  const d = new Date(now);
  if (d.getHours() < 2) d.setDate(d.getDate() - 1);
  const pad = n => String(n).padStart(2, '0');
  const key = `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}`;
  if (typeof TT_HOLIDAYS !== 'undefined' && TT_HOLIDAYS.has(key)) return 'ho';
  const w = d.getDay();
  return (w === 0 || w === 6) ? 'ho' : 'wd';
}

// 直前に出た1本＋次の9本
function pickTrains(rows, nowMin) {
  let nextIdx = rows.findIndex(r => r[0] >= nowMin);
  if (nextIdx < 0) nextIdx = rows.length;
  const start = Math.max(0, nextIdx - 1);
  return { list: rows.slice(start, start + 10), nextIdx: nextIdx - start };
}

function fmtHHMM(min) {
  const pad = n => String(n).padStart(2, '0');
  return `${pad(Math.floor(min / 60) % 24)}:${pad(min % 60)}`;
}

function fmtRemainSec(sec) {
  const pad = n => String(n).padStart(2, '0');
  if (sec <= 0) return '0:00';
  const h = Math.floor(sec / 3600), m = Math.floor((sec % 3600) / 60), s = sec % 60;
  return h > 0 ? `${h}:${pad(m)}:${pad(s)}` : `${m}:${pad(s)}`;
}

// keepScroll が真ならスクロール位置を維持する（1秒ごとの再描画用）
function renderTrackerList(keepScroll) {
  const el = document.getElementById('tracker-list');
  if (!trackerOn) { el.innerHTML = ''; return; }
  const savedScroll = keepScroll ? el.scrollTop : 0;

  const board = (typeof TIMETABLE !== 'undefined') ? TIMETABLE[trackerBoardId] : null;
  if (!board) {
    el.innerHTML = '<div id="tracker-empty">時刻表データがありません</div>';
    return;
  }
  const now    = new Date();
  const rows   = board[trackerDiaKind(now)] || [];
  const nowMin = trackerNowMin(now);
  const picked = pickTrains(rows, nowMin);
  if (picked.list.length === 0) {
    el.innerHTML = '<div id="tracker-empty">本日の運行は終了しました</div>';
    return;
  }

  const delay    = trackerDelayInfo(board.line);
  const nowSecOf = min => (min - nowMin) * 60 - now.getSeconds();

  el.innerHTML = picked.list.map((r, i) => {
    const isNext = (i === picked.nextIdx);
    const isPast = (i < picked.nextIdx);
    const cls    = 'tracker-row' + (isPast ? ' past' : '') + (isNext ? ' next' : '');
    const color  = delay ? ` style="color:${delay.color}"` : '';
    const arr    = r[3] >= 0 ? `${escHtml(board.arrLabel)} ${fmtHHMM(r[3])}` : '';
    const left   = isNext ? fmtRemainSec(nowSecOf(r[0])) : '';
    return `<div class="${cls}"${color}>` +
      `<span class="tr-mark">${isNext ? '▶' : ''}</span>` +
      `<span class="tr-dep">${fmtHHMM(r[0])}</span>` +
      `<span class="tr-bang">${delay ? '!' : ''}</span>` +
      `<span class="tr-type">${escHtml(TT_TYPES[r[1]] || '')}</span>` +
      `<span class="tr-dest">${escHtml(TT_DESTS[r[2]] || '')}</span>` +
      `<span class="tr-arr">${arr}</span>` +
      `<span class="tr-left">${left}</span>` +
      `</div>`;
  }).join('');
  el.scrollTop = savedScroll;
}

// Task 9 で本実装に差し替える
function trackerDelayInfo(lineName) {
  return null;
}
```

- [ ] **Step 4: 1秒ごとの再描画を追加**

既存の `setInterval(updateCountdown, 1000);` の**直後**に、独立した interval を追加する。
既存タイマーの中身は変更しない。

```javascript
setInterval(() => { if (trackerOn && displayMode === 'both') renderTrackerList(true); }, 1000);
```

- [ ] **Step 5: 仮データで表示確認**

データ埋め込み前の確認用に、`index.html` の追跡パネルセクションの直前へ一時的に置く。
**Task 10 で必ず削除する。**

```javascript
// TEMP: Task 10 でデータ埋め込み時に削除
const TT_TYPES = ['各停','急行','快急','特急'];
const TT_DESTS = ['新宿','町田','相模大野'];
const TT_HOLIDAYS = new Set(['2026-01-01']);
const TIMETABLE = {
  G1: { line:'小田急小田原線', arrLabel:'大野',
        wd: [[484,2,0,508],[493,1,0,521],[500,0,1,527],[507,2,0,531],
             [516,1,0,544],[524,3,0,545],[529,2,0,553],[540,1,0,568],
             [549,1,2,577],[555,2,0,579],[566,0,1,594],[577,3,0,598]],
        ho: [[490,2,0,514],[505,0,1,532],[520,1,0,548]] },
  G2: { line:'小田急江ノ島線', arrLabel:'林間', wd: [[500,0,2,505]], ho: [[500,0,2,505]] },
  G3: { line:'東急田園都市線', arrLabel:'南町田', wd: [[510,0,0,514]], ho: [[510,0,0,514]] },
  G4: { line:'東急田園都市線', arrLabel:'終着', wd: [[515,1,0,570]], ho: [[515,1,0,570]] },
  K1: { line:'東急田園都市線', arrLabel:'林間', wd: [[1100,0,0,1105]], ho: [[1100,0,0,1105]] },
  K2: { line:'小田急江ノ島線', arrLabel:'大野', wd: [[1115,0,2,1120]], ho: [[1115,0,2,1120]] },
  K3: { line:'小田急小田原線', arrLabel:'秦野', wd: [[1125,1,0,1158]], ho: [[1125,1,0,1158]] },
  K4: { line:'小田急小田原線', arrLabel:'終着', wd: [[1158,0,1,1190]], ho: [[1158,0,1,1190]] },
};
```

- [ ] **Step 6: ブラウザで表示を確認**

- ALL モード＋追跡 ON で 10行のリストが出る（G1 は12本あるので10行、他は1行）
- 直前の1本がグレーアウトし、次発に `▶` と残り時間が出る
- 残り時間が1秒ごとに減る
- 8ボードすべてボタンで切り替えられる
- 開発者ツールで幅を 375px にしても横スクロールが出ない
- 端末時計を 0:30 に変えると前日ダイヤ（金曜なら `wd`）が使われる
- 天気・カウントダウン・電車カードが変わっていない

- [ ] **Step 7: エンコーディングを検証してコミットコマンドを提示**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
assert b.count(b'\r\n') == b.count(b'\n')
assert b[:3] != b'\xef\xbb\xbf'
b.decode('utf-8'); print('OK', len(b))
"
```

```bash
cd "C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch"
git add index.html _selftest.html
git commit -m "追跡パネルの列車選択ロジックと描画を追加"
```

---

## Task 9: 遅延マークの連動

**Files:**
- Modify: `PROJ/index.html`（`refresh()` に1行、`trackerDelayInfo()` を本実装に）

**Interfaces:**
- Consumes: `refresh()` 内の `lineItems`
- Produces:
  - `lineRankMap` — `{路線名: {rank, color}}`。`refresh()` が更新する
  - `trackerDelayInfo(lineName) -> {rank, color} | null` — ランクが `A` 以外なら情報を返す

- [ ] **Step 1: グローバル変数を追加**

`let isLoading = false;` の**直後**に追加する。

```javascript
// 追跡パネルの遅延マーク用。refresh() が更新する
let lineRankMap = {};
```

- [ ] **Step 2: `refresh()` に1行追加**

`refresh()` 内、`const rankSummary = lineItems.map(...)` の**直前**に追加する。

```javascript
  lineRankMap = Object.fromEntries(lineItems.map(it => [it.line, { rank: it.rank, color: it.rankColor }]));
```

- [ ] **Step 3: `trackerDelayInfo()` の仮実装を差し替える**

```javascript
// 路線ランクが A 以外なら遅延扱い。個々の列車の遅れは見ない
function trackerDelayInfo(lineName) {
  const info = lineRankMap[lineName];
  if (!info || info.rank === 'A' || info.rank === '-') return null;
  return info;
}
```

- [ ] **Step 4: 遅延時の見え方を確認**

一時的に `trackerDelayInfo()` を次のように差し替えてブラウザで確認し、確認後に戻す。

```javascript
function trackerDelayInfo(lineName) { return { rank:'C', color:'#EA4335' }; }
```

- 全行に赤い `!` が出て、行全体がランク色になる
- `!` があっても幅 375px で横スクロールが発生しない
- 直前1本のグレーアウト（`opacity:0.4`）が効いたまま

確認後、Step 3 の実装に戻す。

- [ ] **Step 5: 実データで確認**

- 平常時（全路線ランク A）は `!` が出ない
- ブラウザのコンソールで `lineRankMap` に3路線が入っていることを確認する

```javascript
// コンソールで実行
console.log(lineRankMap);
```

Expected: `小田急小田原線` `小田急江ノ島線` `東急田園都市線` を含むオブジェクト

- [ ] **Step 6: エンコーディング検証とコミットコマンド提示**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
assert b.count(b'\r\n') == b.count(b'\n')
assert b[:3] != b'\xef\xbb\xbf'
b.decode('utf-8'); print('OK', len(b))
"
```

```bash
cd "C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch"
git add index.html
git commit -m "追跡パネルに運行情報ランク連動の遅延マークを追加"
```

---

## Task 10: 実データ埋め込みと総合検証

**Files:**
- Modify: `PROJ/index.html`（TEMP データを実データに差し替え）
- Delete: `PROJ/_selftest.html`
- Modify: `PROJ/SPEC_tracker.md`（実測値の反映）

**Interfaces:**
- Consumes: `WORK/out/timetable.js`
- Produces: 完成した `index.html`

- [ ] **Step 1: TEMP データブロックを実データに差し替える**

Task 8 Step 5 で入れた `// TEMP:` ブロックを削除し、
`WORK/out/timetable.js` の内容をそのまま同じ位置に貼る。

```bash
cd "$PROJ" && python -c "
import re
from pathlib import Path
WORK = r'C:\Users\SHOTAT~1\AppData\Local\Temp\claude\C--Users-ShotaTanaka-Documents--------\7786d944-750b-4f47-bc1a-27177caa3590\scratchpad\tw_build'
# Windows の write_text は \n を \r\n に変換するため、いったん LF に正規化してから CRLF 化する
src = Path(WORK, 'out', 'timetable.js').read_text(encoding='utf-8', newline='')
src = src.replace('\r\n', '\n').replace('\r', '\n').replace('\n', '\r\n')
html = Path('index.html').read_text(encoding='utf-8', newline='')
start = html.index('// TEMP: Task 10')
end   = html.index('};', html.index('const TIMETABLE', start)) + 2
html  = html[:start] + src + html[end:]
Path('index.html').write_text(html, encoding='utf-8', newline='')
print('replaced')
"
```

- [ ] **Step 2: エンコーディングとサイズを検証**

```bash
cd "$PROJ" && python -c "
b=open('index.html','rb').read()
crlf, lf = b.count(b'\r\n'), b.count(b'\n')
print('bytes',len(b),'CRLF',crlf,'LF',lf,'BOM',b[:3]==b'\xef\xbb\xbf')
assert crlf == lf, 'LF単独混入'
assert b[:3] != b'\xef\xbb\xbf', 'BOM混入'
t = b.decode('utf-8')
assert 'TEMP: Task 10' not in t, 'TEMPデータが残っている'
for k in ['G1','G2','G3','G4','K1','K2','K3','K4']:
    assert f'  {k}: {{' in t, f'{k} が無い'
print('OK: 8ボード投入済み / UTF-8 / CRLF only / no BOM')
"
```

Expected: `OK: 8ボード投入済み / UTF-8 / CRLF only / no BOM`

- [ ] **Step 3: 全ボードをブラウザで確認**

8ボードすべてについて、Yahoo の時刻表ページと突き合わせる。

| 確認項目 | 内容 |
|---|---|
| 発時刻 | 現在時刻前後の3本が Yahoo と一致 |
| 種別 | 各停/急行/快急/特急 の表記が正しい |
| 行先 | 無印の列車が方面の代表駅になっている |
| 到着 | 到着時刻が発時刻より後 |
| 平日/土休日 | 端末日付を土曜に変えると本数が変わる |

- [ ] **Step 4: デグレチェック（`/yokoten-check` 相当）**

改修前の `index.html`（`git stash` せず `git show HEAD:index.html` で取得）と比較し、
既存機能に差分が無いことを確認する。

```bash
cd "$PROJ" && git show HEAD:index.html > /tmp/before.html 2>/dev/null || git show HEAD:index.html > before_ref.html
```

確認する既存機能の一覧。

- [ ] 時計が1秒ごとに更新される
- [ ] 日付・曜日が正しい
- [ ] バッテリー表示（`⚡` と色分け）
- [ ] `↺` 手動更新でスピナーが回り、天気と電車情報が再取得される
- [ ] 天気モード: 傘行・2地点の今日/明日が出る。カウントダウンは出ない
- [ ] CD モード: カウントダウンのみ。天気・NTP が出ない
- [ ] ALL モード: カウントダウン＋天気＋追跡パネル
- [ ] モードセレクトが 天気 → CD → ALL → 天気 で循環する
- [ ] モードがリロード後も復元される
- [ ] 任意時刻ボタンが CD/ALL でのみ出る。入力すると反映される
- [ ] ランクサマリー行（`A / A / A / A- / C`）が出る
- [ ] 広域カードと監視5路線カードが出る
- [ ] NTP インジケータが `未取得` → `更新中` → `OK` と変わる
- [ ] 追跡 OFF のとき、追跡パネルの1秒タイマーが何もしない

- [ ] **Step 5: `_selftest.html` を削除**

```bash
cd "$PROJ" && rm _selftest.html && ls
```

Expected: `SPEC.md  SPEC_tracker.md  bash.exe.stackdump  index.html`

- [ ] **Step 6: `SPEC_tracker.md` に実測値を反映**

以下を確定値に更新する。

- セクション3.3: 土曜と日祝が一致したかどうかの結果
- セクション4.2: 実際のリクエスト数と所要時間
- セクション4.3: 確定した固定所要時間（`WORK/out/calibration.md` から転記）
- セクション3.5: 実際のデータ量と `index.html` の最終サイズ

- [ ] **Step 7: サブエージェントで結論を検証（CLAUDE.md ルール18）**

サブエージェントに以下を渡し、**結論を否定する材料を探させる**。

- `PROJ/index.html`（改修後）
- `PROJ/SPEC_tracker.md`
- `git show HEAD:index.html`（改修前）
- 「追跡パネルの追加によって既存機能にデグレが無い」という結論

指摘があれば元データに戻って再確認し、結論を直す。
報告では「裏取り済み」と「未確認（実機・実運用でのみ確認可能なもの）」を分けて書く。

- [ ] **Step 8: 最終コミットコマンドを提示（実行しない）**

```bash
cd "C:\Users\ShotaTanaka\Documents\work\OLDproject\TrainWatch"
git add index.html SPEC_tracker.md
git status
git commit -m "TrainWatcher: ALLモードに通勤4駅の追跡パネルを追加"
```

---

## 完了条件

`SPEC_tracker.md` セクション6の受け入れ条件をすべて満たすこと。

- [ ] ALL モード以外では追跡パネルが表示されない
- [ ] 追跡 ON/OFF がリロード後も復元される
- [ ] 2:00〜11:59 で「行き・秦野」、12:00〜1:59 で「帰り・中央林間」が選ばれる
- [ ] 8ボードすべてが表示でき、発時刻・種別・行先・到着時刻が出る
- [ ] 直前1本がグレーアウトし、次発に `▶` と残り時間が出る
- [ ] 平日・土休日がカレンダーに応じて切り替わる（祝日含む）
- [ ] 0時台の列車が当日末尾に並び、0〜2時に開いても破綻しない
- [ ] 該当路線のランクが A 以外のとき全行に赤い `!` とランク色が付く
- [ ] 幅375px で横スクロールが発生しない
- [ ] 既存機能の表示・挙動が改修前と同一
- [ ] `index.html` が UTF-8(BOMなし)・CRLF のまま
- [ ] `_selftest.html` が削除されている
