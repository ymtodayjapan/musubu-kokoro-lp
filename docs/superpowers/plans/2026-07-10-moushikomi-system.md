# 「世界と話そう」申し込み管理システム 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** LPの各コースに申し込みボタンを設置し、Googleフォーム→自動振込案内メール→入金管理→当日出席リスト連動までを1本のGASスクリプトで構築する。

**Architecture:** 静的LP（Vercel/Cloudflare自動デプロイ）からGoogleフォームへリンク。フォームはスプレッドシートに連携し、GASのonFormSubmit/onEditトリガーで自動メールとリスト管理を行う。口座番号を含むファイルはGit管理外の `private/` に置く（リポジトリが公開のため）。

**Tech Stack:** HTML/CSS（既存LP）、Google Apps Script（V8）、Googleフォーム、Googleスプレッドシート

**重要な制約:**
- `git push` すると Cloudflare／Vercel に自動デプロイされる。LPの変更はフォームURL差し替え完了までプッシュしない。
- リポジトリは公開。口座番号は `private/` 内のファイルにのみ記載し、`.gitignore` で除外する。
- 計画書中のGASコードでは口座番号を `【口座番号】` と伏せてある。private版作成時に実際の番号（チャットで受領済み）へ置き換える。

---

### Task 1: private/ フォルダをGit管理外にする

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: .gitignore に private/ を追加**

```
.DS_Store
private/
```

- [ ] **Step 2: 動作確認**

Run: `mkdir -p private && touch private/test.txt && git status --short && rm private/test.txt`
Expected: 出力に `private/` が現れない（`.gitignore` の変更のみ表示される）

- [ ] **Step 3: コミット**

```bash
git add .gitignore
git commit -m "private/ をGit管理外に追加（口座情報を含むファイル用）"
```

---

### Task 2: GASスクリプト本体を作成

**Files:**
- Create: `private/gas-moushikomi-system.gs`

1本のスクリプトに「セットアップ」「申込時の自動メール」「入金チェック時の処理」をすべて含める。よしこさんは script.google.com に貼り付けて `setup` を1回実行するだけ。

- [ ] **Step 1: スクリプト全文を作成**（口座番号は実際の値に置き換えて保存する）

```javascript
/**
 * 「世界と話そう〜日本を伝えよう〜」申し込み管理システム
 * セットアップ方法：setup() を1回だけ実行してください。
 */

// ====== 設定 ======
const CONFIG = {
  ADMIN_EMAIL: 'ym.todayjapan@gmail.com',
  SENDER_NAME: 'Today Japan 増田美子',
  FORM_TITLE: '「世界と話そう〜日本を伝えよう〜」お申し込みフォーム',
  SPREADSHEET_TITLE: '世界と話そう 申込管理',
  COURSES: {
    '親子セット（20,000円）': 20000,
    '単独参加（15,000円）': 15000,
    '当日参加（5,000円）': 5000,
  },
  CALENDAR_CHOICES: [
    '追加なし',
    '1冊（+1,500円）',
    '2冊（+3,000円）',
    '3冊（+4,500円）',
    '4冊（+6,000円）',
    '5冊（+7,500円）',
  ],
  LEVELS: [
    '初級（これから楽しく始めたい）',
    '中級（かんたんな会話ならできる）',
    '上級（英語でのやりとりに慣れている）',
  ],
  CALENDAR_PRICE: 1500,
  PAYMENT_DUE_DAYS: 7,
  BANK_INFO:
    '三井住友銀行　名古屋支店（481）\n' +
    '普通　【口座番号】\n' +
    '口座名義：TodayJapan株式会社\n' +
    '　　　　　トウデイジヤパンカブシキカイシヤ',
};

// 申込リストの列番号（A=1）
const COL = {
  TIMESTAMP: 1, NAME: 2, FURIGANA: 3, EMAIL: 4, PHONE: 5,
  COURSE: 6, KODOMO: 7, LEVEL: 8, CALENDAR: 9, COMMENT: 10,
  TOTAL: 11, DUE: 12, PAID: 13, PAID_DATE: 14, PAID_MAIL: 15, MEMO: 16,
};

const SHEET_MOUSHIKOMI = '申込リスト';
const SHEET_SHUSSEKI = '当日出席リスト';
const SHEET_SHUKEI = '集計';

// ====== セットアップ（1回だけ実行） ======
function setup() {
  // 1. フォーム作成
  const form = FormApp.create(CONFIG.FORM_TITLE);
  form.setDescription(
    '「世界と話そう〜日本を伝えよう〜」へのお申し込みフォームです。\n' +
    '送信後、ご記入のメールアドレスへ「お振込み先のご案内」を自動でお送りします。\n' +
    '届かない場合は、迷惑メールフォルダもご確認くださいね。'
  );
  form.setConfirmationMessage(
    'お申し込みありがとうございます！\n' +
    'ご記入のメールアドレスへ、お振込み先のご案内をお送りしました。\n' +
    '届かない場合は迷惑メールフォルダをご確認ください。\n\n' +
    '当日お会いできるのを楽しみにしています🌏'
  );

  form.addTextItem().setTitle('お名前（保護者の方）').setRequired(true);
  form.addTextItem().setTitle('ふりがな').setRequired(true);

  const emailItem = form.addTextItem().setTitle('メールアドレス').setRequired(true);
  emailItem.setValidation(
    FormApp.createTextValidation()
      .requireTextIsEmail()
      .setHelpText('メールアドレスの形式で入力してください')
      .build()
  );

  form.addTextItem().setTitle('電話番号').setRequired(true);

  const courseItem = form.addMultipleChoiceItem()
    .setTitle('参加コース')
    .setChoiceValues(Object.keys(CONFIG.COURSES))
    .setRequired(true);

  form.addParagraphTextItem()
    .setTitle('お子さんのお名前と年齢（親子セットの方）')
    .setHelpText('例：はな（8歳）、たろう（5歳）');

  form.addMultipleChoiceItem()
    .setTitle('英語レベル')
    .setHelpText('いまの気持ちに近いものを選んでください。どのレベルでも大丈夫です😊')
    .setChoiceValues(CONFIG.LEVELS)
    .setRequired(true);

  form.addListItem()
    .setTitle('カレンダー追加購入（1冊1,500円）')
    .setChoiceValues(CONFIG.CALENDAR_CHOICES)
    .setRequired(true);

  form.addParagraphTextItem()
    .setTitle('フリーコメント（ご質問・メッセージなど、何でもどうぞ）');

  // 2. スプレッドシート作成・連携
  const ss = SpreadsheetApp.create(CONFIG.SPREADSHEET_TITLE);
  form.setDestination(FormApp.DestinationType.SPREADSHEET, ss.getId());
  SpreadsheetApp.flush();

  // 回答シートを「申込リスト」にリネーム
  const ss2 = SpreadsheetApp.openById(ss.getId());
  const responseSheet = ss2.getSheets().filter(function (s) {
    return s.getFormUrl();
  })[0];
  responseSheet.setName(SHEET_MOUSHIKOMI);

  // 管理用の列を追加
  const headers = ['合計金額', '振込期限', '入金済み', '入金日', '入金確認メール', 'メモ'];
  responseSheet.getRange(1, COL.TOTAL, 1, headers.length).setValues([headers]).setFontWeight('bold');
  responseSheet.getRange(2, COL.PAID, 999, 1).insertCheckboxes();

  // 当日出席リスト
  const shusseki = ss2.insertSheet(SHEET_SHUSSEKI);
  shusseki.getRange(1, 1, 1, 7).setValues([[
    'お名前', 'ふりがな', 'コース', 'お子さんのお名前と年齢', '英語レベル', 'カレンダー追加', '出席',
  ]]).setFontWeight('bold');

  // 集計
  const shukei = ss2.insertSheet(SHEET_SHUKEI);
  shukei.getRange(1, 1, 1, 5).setValues([[
    'コース', '申込数', '入金済み', '売上見込み', '入金済み金額',
  ]]).setFontWeight('bold');
  const courses = Object.keys(CONFIG.COURSES);
  for (let i = 0; i < courses.length; i++) {
    const r = i + 2;
    shukei.getRange(r, 1).setValue(courses[i]);
    shukei.getRange(r, 2).setFormula('=COUNTIF(' + SHEET_MOUSHIKOMI + '!F:F, A' + r + ')');
    shukei.getRange(r, 3).setFormula('=COUNTIFS(' + SHEET_MOUSHIKOMI + '!F:F, A' + r + ', ' + SHEET_MOUSHIKOMI + '!M:M, TRUE)');
    shukei.getRange(r, 4).setFormula('=SUMIF(' + SHEET_MOUSHIKOMI + '!F:F, A' + r + ', ' + SHEET_MOUSHIKOMI + '!K:K)');
    shukei.getRange(r, 5).setFormula('=SUMIFS(' + SHEET_MOUSHIKOMI + '!K:K, ' + SHEET_MOUSHIKOMI + '!F:F, A' + r + ', ' + SHEET_MOUSHIKOMI + '!M:M, TRUE)');
  }
  const totalRow = courses.length + 2;
  shukei.getRange(totalRow, 1).setValue('合計').setFontWeight('bold');
  for (let c = 2; c <= 5; c++) {
    const colLetter = String.fromCharCode(64 + c);
    shukei.getRange(totalRow, c).setFormula(
      '=SUM(' + colLetter + '2:' + colLetter + (totalRow - 1) + ')'
    ).setFontWeight('bold');
  }

  // 3. トリガー設置（既存トリガーは一度すべて削除）
  ScriptApp.getProjectTriggers().forEach(function (t) { ScriptApp.deleteTrigger(t); });
  ScriptApp.newTrigger('onFormSubmitHandler').forSpreadsheet(ss2).onFormSubmit().create();
  ScriptApp.newTrigger('onPaidEditHandler').forSpreadsheet(ss2).onEdit().create();

  // 4. コース別の事前入力済みURLを出力
  const lines = [];
  lines.push('========== セットアップ完了！ ==========');
  lines.push('');
  lines.push('▼ スプレッドシート（申込管理）');
  lines.push(ss2.getUrl());
  lines.push('');
  lines.push('▼ フォーム編集画面');
  lines.push(form.getEditUrl());
  lines.push('');
  lines.push('▼ 申し込みフォームURL（コース選択なし・基本URL）');
  lines.push(form.getPublishedUrl());
  lines.push('');
  Object.keys(CONFIG.COURSES).forEach(function (course) {
    const resp = form.createResponse();
    resp.withItemResponse(courseItem.createResponse(course));
    lines.push('▼ ' + course + ' 用URL');
    lines.push(resp.toPrefilledUrl());
    lines.push('');
  });
  lines.push('この内容をコピーして、Claudeに送ってください。');
  const result = lines.join('\n');
  Logger.log(result);

  // 実行ログを見なくても済むよう、自分宛にもメールする
  GmailApp.sendEmail(CONFIG.ADMIN_EMAIL, '【世界と話そう】セットアップ完了・フォームURL一覧', result, {
    name: CONFIG.SENDER_NAME,
  });
}

// ====== 申し込み時の処理 ======
function onFormSubmitHandler(e) {
  const sheet = e.range.getSheet();
  const row = e.range.getRow();
  const v = e.namedValues;
  const pick = function (key) { return (v[key] && v[key][0]) ? v[key][0] : ''; };

  const name = pick('お名前（保護者の方）');
  const email = pick('メールアドレス');
  const course = pick('参加コース');
  const calChoice = pick('カレンダー追加購入（1冊1,500円）');
  const level = pick('英語レベル');

  // 合計金額の計算
  const coursePrice = CONFIG.COURSES[course] || 0;
  const calCount = parseInt(calChoice, 10) || 0; // 「追加なし」は0になる
  const total = coursePrice + calCount * CONFIG.CALENDAR_PRICE;

  // 振込期限（申込日 + 7日）
  const due = new Date();
  due.setDate(due.getDate() + CONFIG.PAYMENT_DUE_DAYS);
  const dueStr = formatJaDate(due);

  sheet.getRange(row, COL.TOTAL).setValue(total);
  sheet.getRange(row, COL.DUE).setValue(dueStr);

  // 申込者への自動メール
  const body =
    name + ' さま\n\n' +
    'このたびは「世界と話そう〜日本を伝えよう〜」に\n' +
    'お申し込みいただき、ありがとうございます！\n' +
    'ご一緒できることが、今からとても楽しみです😊\n\n' +
    '■ お申し込み内容\n' +
    '・参加コース：' + course + '\n' +
    '・カレンダー追加：' + calChoice + '\n' +
    '・英語レベル：' + level + '\n' +
    '・合計金額：' + formatYen(total) + '円\n\n' +
    '■ お振込み先\n' +
    CONFIG.BANK_INFO + '\n\n' +
    '■ お振込み期限\n' +
    dueStr + '（お申し込みから' + CONFIG.PAYMENT_DUE_DAYS + '日以内）\n\n' +
    '※ 恐れ入りますが、振込手数料はご負担くださいますよう\n' +
    '　お願い申し上げます。\n\n' +
    'ご入金を確認しましたら、あらためてメールでお知らせしますね。\n' +
    'ご不明な点があれば、このメールにそのままご返信ください。\n\n' +
    '世界はもっと、仲良くなれる。\n' +
    '当日お会いできるのを楽しみにしています🌏\n\n' +
    '─────────────────\n' +
    'Today Japan株式会社　増田美子\n' +
    'https://todayjapan.net\n' +
    '─────────────────';

  try {
    GmailApp.sendEmail(email, '【世界と話そう】お申し込みありがとうございます（お振込みのご案内）', body, {
      name: CONFIG.SENDER_NAME,
    });
  } catch (err) {
    sheet.getRange(row, COL.MEMO).setValue('メール送信エラー：' + err.message);
    notifyAdminSafe('【要確認】自動メールの送信に失敗しました',
      name + 'さまへの自動メール送信に失敗しました。\nメールアドレス：' + email +
      '\nエラー：' + err.message + '\nスプレッドシートから直接ご連絡ください。');
  }

  // よしこさんへの通知
  notifyAdminSafe('【新しいお申し込み】' + name + ' さま（' + course + '）',
    '新しいお申し込みが入りました！\n\n' +
    '・お名前：' + name + '\n' +
    '・コース：' + course + '\n' +
    '・カレンダー追加：' + calChoice + '\n' +
    '・合計金額：' + formatYen(total) + '円\n' +
    '・振込期限：' + dueStr + '\n\n' +
    'スプレッドシート「' + CONFIG.SPREADSHEET_TITLE + '」でご確認ください。');
}

// ====== 入金済みチェック時の処理 ======
function onPaidEditHandler(e) {
  const sheet = e.range.getSheet();
  if (sheet.getName() !== SHEET_MOUSHIKOMI) return;
  if (e.range.getColumn() !== COL.PAID || e.range.getNumRows() > 1 || e.range.getNumColumns() > 1) return;
  const row = e.range.getRow();
  if (row < 2) return;
  if (e.range.getValue() !== true) return; // チェックを入れた時だけ

  const values = sheet.getRange(row, 1, 1, COL.MEMO).getValues()[0];
  const name = values[COL.NAME - 1];
  const furigana = values[COL.FURIGANA - 1];
  const email = values[COL.EMAIL - 1];
  const course = values[COL.COURSE - 1];
  const kodomo = values[COL.KODOMO - 1];
  const level = values[COL.LEVEL - 1];
  const calChoice = values[COL.CALENDAR - 1];
  const alreadySent = values[COL.PAID_MAIL - 1] === '送信済み';

  if (alreadySent) return; // 二重送信・二重追加の防止

  sheet.getRange(row, COL.PAID_DATE).setValue(formatJaDate(new Date()));

  // 入金確認メール
  const body =
    name + ' さま\n\n' +
    'ご入金を確認しました。\n' +
    'ありがとうございます！\n\n' +
    'これで参加のお手続きは完了です✨\n' +
    'オンライン会や当日のご案内など、最新情報は\n' +
    'オープンチャット「🌏世界と話そう🇯🇵」でお届けしていきます。\n\n' +
    '当日、お会いできるのを楽しみにしています。\n' +
    '今日もあなたはサイコーです😊\n\n' +
    '─────────────────\n' +
    'Today Japan株式会社　増田美子\n' +
    'https://todayjapan.net\n' +
    '─────────────────';

  try {
    GmailApp.sendEmail(email, '【世界と話そう】ご入金を確認しました', body, {
      name: CONFIG.SENDER_NAME,
    });
    sheet.getRange(row, COL.PAID_MAIL).setValue('送信済み');
  } catch (err) {
    sheet.getRange(row, COL.MEMO).setValue('入金確認メール送信エラー：' + err.message);
    notifyAdminSafe('【要確認】入金確認メールの送信に失敗しました',
      name + 'さまへの入金確認メール送信に失敗しました。\nエラー：' + err.message);
  }

  // 当日出席リストに追加
  const ss = sheet.getParent();
  const shusseki = ss.getSheetByName(SHEET_SHUSSEKI);
  shusseki.appendRow([name, furigana, course, kodomo, level, calChoice, false]);
  shusseki.getRange(shusseki.getLastRow(), 7).insertCheckboxes();
}

// ====== 共通の小道具 ======
function formatJaDate(d) {
  const youbi = ['日', '月', '火', '水', '木', '金', '土'][d.getDay()];
  return (d.getMonth() + 1) + '月' + d.getDate() + '日（' + youbi + '）';
}

function formatYen(n) {
  return String(n).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
}

function notifyAdminSafe(subject, body) {
  try {
    GmailApp.sendEmail(CONFIG.ADMIN_EMAIL, subject, body, { name: CONFIG.SENDER_NAME });
  } catch (err) {
    // 通知メールの失敗は握りつぶす（本処理を止めない）
  }
}
```

- [ ] **Step 2: 構文チェック**

Run: `cp private/gas-moushikomi-system.gs /tmp-scratch/check.js && node --check /tmp-scratch/check.js`（scratchpad配下で実行）
Expected: エラーなし（終了コード0）

- [ ] **Step 3: 伏せ字が残っていないか確認**

Run: `grep -c '【口座番号】' private/gas-moushikomi-system.gs`
Expected: `0`（実際の口座番号に置き換え済み）

コミットはしない（private/ はGit管理外）。

---

### Task 3: セットアップ手順書を作成

**Files:**
- Create: `private/moushikomi-setup-guide.md`

よしこさん向け。以下の構成で、専門用語を使わず1手順1操作で書く：

- [ ] **Step 1: 手順書を作成**

内容（章立て）：
1. **これは何？** — 仕組みの全体図（申し込み→自動メール→入金チェック→出席リスト）
2. **セットアップ（最初に1回だけ）**
   - script.google.com を開く →「新しいプロジェクト」
   - 最初から入っているコードを全部消して、`gas-moushikomi-system.gs` の中身を貼り付け
   - プロジェクト設定（歯車マーク）でタイムゾーンが「(GMT+09:00) 東京」になっているか確認
   - 「setup」を選んで実行 → 権限の許可画面の進み方（「詳細」→「安全ではないページに移動」の説明を含む）
   - 完了するとGmailに「セットアップ完了・フォームURL一覧」メールが届く → その内容をClaudeに送る
   - ※ setup は2回実行しないこと（フォームが2つできてしまう）
3. **ふだんの使い方**
   - 申し込みが入ると：Gmailに通知＋申込リストに自動で1行追加
   - 入金を確認したら：申込リストの「入金済み」にチェック → 自動でお礼メール＋出席リストに追加
   - 間違えてチェックした場合：チェックを外し、出席リストの行を右クリック→「行を削除」
   - 当日：「当日出席リスト」タブを開いて「出席」にチェックするだけ
4. **困ったとき** — メールが届かない（迷惑メール確認・1日100通の上限）、金額を変えたいとき（CONFIGの説明）

- [ ] **Step 2: 口座情報（実物）が正しく記載されているか目視確認**

コミットはしない（private/ はGit管理外）。

---

### Task 4: LPに申し込みボタンを追加

**Files:**
- Modify: `sekai-to-hanasou.html`（3つのプランカード＋「お申込みについて」セクション）

プレースホルダURL（後で差し替え）：
- `FORM_URL_OYAKO` — 親子セット用
- `FORM_URL_TANDOKU` — 単独参加用
- `FORM_URL_TOJITSU` — 当日参加用
- `FORM_URL_BASE` — コース未選択の基本URL（お申込みセクション用）

- [ ] **Step 1: ボタン用CSSを追加**（既存の `.oc-btn` の近くに）

```css
.plan-apply{
  display:block;text-align:center;margin-top:14px;
  background:#fff;border-radius:999px;
  font-size:14px;font-weight:800;text-decoration:none;
  padding:11px 10px;letter-spacing:.02em;
  box-shadow:0 3px 10px rgba(58,54,50,.12);
  transition:transform .15s;
}
.plan-apply:active{transform:scale(.97);}
.plan.a .plan-apply{color:#e2647f;border:2px solid #f5849b;}
.plan.b .plan-apply{color:#2b8fc6;border:2px solid #3fa9e0;}
.plan.c .plan-apply{color:#3da878;border:2px solid #68c69a;}
```

- [ ] **Step 2: 各プランカードの `</ul>` の後にボタンを追加**

親子セット（`.plan.a` の plan-body 内）：
```html
<a class="plan-apply" href="FORM_URL_OYAKO" target="_blank" rel="noopener">🌸 親子セットで申し込む</a>
```
単独参加（`.plan.b`）：
```html
<a class="plan-apply" href="FORM_URL_TANDOKU" target="_blank" rel="noopener">🌊 単独参加で申し込む</a>
```
当日参加（`.plan.c`）：
```html
<a class="plan-apply" href="FORM_URL_TOJITSU" target="_blank" rel="noopener">🌱 当日参加で申し込む</a>
```

- [ ] **Step 3: 「お申込みについて」セクションを申込受付中の内容に書き換え**

apply-card の中身を差し替え：
```html
<div class="apply-card">
  <span class="apply-badge">申込受付中</span>
  <p class="big">お申し込みは、<br>各コースのボタンからどうぞ ✨</p>
  <p>
    お申し込みが完了すると、<br>
    お振込み先のご案内メールが自動で届きます。<br>
    ご入金の確認ができましたら、<br>
    完了メールをお送りしますね 😊
  </p>
  <a class="oc-btn" href="FORM_URL_BASE" target="_blank" rel="noopener">📝 申し込みフォームを開く</a>
  <p style="margin-top:18px;">
    最新情報はオープンチャット<br>「🌏世界と話そう🇯🇵」でもお届けしています。
  </p>
  <a class="oc-btn" href="https://line.me/ti/g2/ODaFXHxXlZK-T6jcaipI6hjEu5ms-R3MLmFw0g?utm_source=invitation&amp;utm_medium=link_copy&amp;utm_campaign=default" target="_blank" rel="noopener">💬 オープンチャットに参加する</a>
</div>
```

- [ ] **Step 4: ブラウザプレビューで確認**

プレビューで3つのボタンの表示・配色・スマホ幅での見え方を確認。
Expected: 各カードにボタンが表示され、レイアウト崩れがない。

- [ ] **Step 5: コミット（プッシュしない）**

```bash
git add sekai-to-hanasou.html
git commit -m "世界と話そう：各コースに申し込みボタンを追加（フォームURLは差し替え待ち）"
```

**注意：このタスクの後、`git push` しないこと。** プレースホルダのままデプロイされてしまう。

---

### Task 5: フォームURL差し替えとデプロイ（よしこさんのsetup実行後）

**Files:**
- Modify: `sekai-to-hanasou.html`

- [ ] **Step 1: よしこさんからURL一覧（setup完了メールの内容）を受け取る**
- [ ] **Step 2: 4つのプレースホルダを実URLに置換**
- [ ] **Step 3: 残置確認**

Run: `grep -c 'FORM_URL' sekai-to-hanasou.html`
Expected: `0`

- [ ] **Step 4: コミット＆プッシュ（自動デプロイ）**

```bash
git add sekai-to-hanasou.html
git commit -m "世界と話そう：申し込みフォームURLを設定し公開"
git push
```

- [ ] **Step 5: 公開URLで3ボタンからフォームが正しいコース選択済みで開くことを確認**

---

## テスト（よしこさんと一緒に行う受け入れ確認）

1. テスト申し込み（自分のメールアドレスで各コース1件、カレンダー追加あり）→ 自動メールの文面・金額・期限日付を確認
2. 「入金済み」チェック → 入金確認メール受信・出席リストに行追加を確認
3. チェック→外す→再チェックで、メールが二重送信されないことを確認（「入金確認メール」列が送信済みならスキップ）
4. 集計タブの数字がテストデータと一致することを確認
5. テストデータを申込リスト・出席リストから削除
