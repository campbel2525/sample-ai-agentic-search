# 概要

最近この記事をみました

- なぜ、Claude Codeは、RAGを捨ててAgentic Searchを選んだのか？
  - https://zenn.dev/karamage/articles/2514cf04e0d1ac

この記事では

- ベクトルDBを使用せずAI エージェントがgrepコマンドを使用してRAGする(エージェンティック サーチ)

と書かれていました。

正しいテキストファイルが用意されているのであれば、確かにこの方法でRAGはできそうと感じました。

また今回は用意されているAIエージェントを実行するだけなので簡単にできそうです

以前AIエージェント + ベクトル検索を作ってみたことがあるのですが、精度は良くありませんでした。特にプロンプトの調整が難しかったです

- https://github.com/campbel2525/sample-ai-agent-rag

フォルダ構成

```bash
├── .gitignore
├── Dockerfile.ai-agent
├── README.md
├── docker-compose.yml
├── docs                           # 元PDF置き場
│   ├── MB20240318_2page.pdf
│   └── モデル就業規則.pdf
├── project                        # 作業ディレクトリ （コンテナ内で /project にマウント）
│   ├── CLAUDE.md                  # Claude Code向けのプロジェクト指示（ローカル用・gitignore対象）
│   ├── CLAUDE.md.example          # `CLAUDE.md` の雛形（現状空）
│   └── files                      # 検索/RAG用に置く素材
│       ├── images                 # PDFをページ画像化したもの
│       │   ├── MB20240318_2page   # `1.png`, `2.png`
│       │   └── モデル就業規則       # `1.png`〜`89.png`
│       ├── ocr                    # 画像のOCR結果（Markdown）
│       │   ├── MB20240318_2page   # `1.md`, `2.md`
│       │   └── モデル就業規則       # `1.md`〜`89.md`
│       └── pdf                    # PDFのコピー置き場（現状空）
└── settings                       # 各CLIの設定ファイル（ホスト側で管理→composeでマウント）
    ├── .claude                    # Claude Code設定
    │   ├── settings.json          # env設定（例: Agent Teams有効化）
    │   └── settings.json.example  # 雛形
    └── .codex                     # Codex CLI設定
        ├── AGENTS.md              # エージェントへの指示（例: 日本語で回答）
        ├── AGENTS.md.example      # 雛形
        ├── config.toml            # モデル/承認ポリシー/サンドボックス等の設定
        └── config.toml.example    # 雛形（Playwright MCP設定例つき）
```

## RAGについてのまとめ

RAGについて調べてみるといくつか定義がありました。

- 従来のRAG
  - ベクターDBを使用したRAG
- エージェンティックRAG
  - AIエージェント + ベクターDBを使用したRAG
- エージェンティックサーチ
  - ベクターDBを使用せずAIエージェントがgrepなどを駆使して検索を行う
  - 今回はこちらを試してみました

## 構築した内容

厚労省の就業に関するPDF(89ページ)についてエージェンティックサーチを実行するサンプルを作りました。

エージェントには

- codex
- claude code
- claude code エージェントチーム

を使用しました。これらは月額のプランで使用できるため結構リーズナブルかと思います。

プラン

- codex: 月額3000円
- claude: 月額3400円

またcodexやcalude codeはdocker環境で動かすことを想定しています。

# 環境構築

1. 設定ファイルの作成

- `settings/.claude/settings.json.example`を参考にして`settings/.claude/settings.json`を作成
- `settings/.codex/AGENTS.md.example`を参考にして`settings/.codex/AGENTS.md`を作成
- `settings/.codex/config.toml.example`を参考にして`settings/.codex/config.toml`を作成
- `CLAUDE.md.example`を参考にして`project/CLAUDE.md`を作成

1. docker環境の構築

```bash
docker compose up -d --build
```

1. codex, claude codeのログイン

各コマンドを実行適切に認証を行う。認証が終わったら`ctl + C`で抜けてOK

```bash
# codexの起動
docker compose exec ai-agent codex
# 月額プランで実行する際の認証方法は
# Sign in with Device Codeで認証を勧めた方がいいです

# calude codeの起動
docker compose exec ai-agent claude
```

1. codex, claude codeの実行。

以下のコマンドを実行すると立ち上がります

```bash
# codexの起動
docker compose exec ai-agent codex

# calude codeの起動
docker compose exec ai-agent claude
```

# Tips

1. いちいち許可しないで自動で実行してくれる起動コマンド。ただしrootユーザーだとこのコマンドが実行できないため、Dockerfileではnodeユーザーで実行するように変更してあります

   ```
   claude --dangerously-skip-permissions
   ```

2. codexやclaude codeではPDF単体ではOCRできないので、いったん画像に変換してから画像をOCRするといい
3. codexは3000プラン、claude: 3400プランですが、codexは3000プランの方がかなり回数が使用できる。claudeの方はすぐにリミットに達してしまう

# 参考文献

- Claude Code のオーケストレーション機能であるエージェントチームを試してみた
  - https://azukiazusa.dev/blog/trying-claude-code-agent-teams/
- なぜ、Claude Codeは、RAGを捨ててAgentic Searchを選んだのか？
  - https://zenn.dev/karamage/articles/2514cf04e0d1ac

# 検証

## 対象PDF

[モデル就業規則.pdf](attachment:48f219cd-8f1c-4599-88de-40206190188e:モデル就業規則.pdf)

## 検証方法

1. PDFをclaude codeを使用しOCRを行う

プロンプトは失念してしまいましたが`PDFを画像化して`といった本当に簡単なものでした

1. 画像のOCRを行う

```
- 画像をOCRしてfiles/ocr/<pdf名><ページ数>.mdとして文字起こしして。マークダウンで文字起こししてね
- 画像をOCRしてfiles/ocr/<pdf名><ページ数>.mdとして文字起こしして。マークダウンで文字起こししてね
```

1. 質問を行う

使用したモデル

- codex
  - gpt-5.2 Extra high
- claude code
  - Opus 4.6

検証で使用したプロンプト

```
あなたはRAGの専門家です

回答する際に知っている知識や常識などは使用しないで`files/ocr/`配下の情報を使って回答してください

# 回答フォーマット

- 回答
- どのPDFか
- どのページ数か
  - 該当箇所の抜粋

# ユーザーの質問

2つの質問にそれぞれ回答してください。回答フォーマットは必ず守ること

- 就業規則はどんな法律？
- 会社の物品を会社とは関係ないところで使用していいの？
```

laudeのエージェントチームを利用したい場合

エージェントチームの実行はプロンプトで行うようです

```bash
あなたはRAGの専門家です。
エージェントチームを使用してタスクを行なってください
回答する際に知っている知識や常識などは使用しないで`files/ocr/`配下の情報を使って回答してください

- 調査エージェント
- 回答エージェント
- 最終回答エージェント

# 回答フォーマット

- 回答
- どのPDFか
- どのページ数か
  - 該当箇所の抜粋

# ユーザーの質問

2つの質問にそれぞれ回答してください。回答フォーマットは必ず守ること

- 就業規則はどんな法律？
- 会社の物品を会社とは関係ないところで使用していいの？
```

回答で使用されるファイル

- 就業規則はどんな法律？
  - p2, p3, p10
  - 労基法」第８９条
- 会社の物品を会社とは関係ないところで使用していいの？
  - p16, 81
  - 第11条 許可なく職務以外の目的で会社の施設、物品等を使用しないこと。

## 実行結果

### codex

回答

```
【質問1】就業規則はどんな法律？

- 回答：就業規則（規則）は「労働基準法（労基法）第89条に基づき、会社の労働者の就業に関する事項を定めるもの」とされています。
- どのPDFか：モデル就業規則
- どのページ数か：10ページ
    - 該当箇所の抜粋：「この就業規則（以下「規則」という。）は、労働基準法（以下「労基法」という。）第89条に基づき、】【株式会社の労働者の就業に関する事項を定める
    ものである。」

【質問2】会社の物品を会社とは関係ないところで使用していいの？

- 回答：「許可なく職務以外の目的で会社の施設、物品等を使用しないこと」と定められています。
- どのPDFか：モデル就業規則
- どのページ数か：16ページ
    - 該当箇所の抜粋：「許可なく職務以外の目的で会社の施設、物品等を使用しないこと。」
```

実行中のエビデンス

```
• Explored
  └ Search 労働基準法 in モデル就業規則
    Search 物品\|備品\|私用\|私的\|業務外\|会社.*使用 in モデル就業規則
    Read 10.md, 16.md

• I see that the policy clearly states not to use company facilities or goods for non-duty purposes without permission. My answer here is simple: it’s not allowed
  without prior approval. I might also want to check 81.md regarding discipline to ensure I have all the relevant information. Let’s take a look at that specific
  section and see if there’s anything more we need to consider before moving forward!

• Explored
  └ Read 81.md

• To ensure accuracy, I'll check if the first line of 10.md corresponds to page 10. I shouldn't assume, but I can treat 10.md as page 10, located in the folder 'モデ
  Searching for work rules definitions

  I need to be careful not to use any external knowledge while answering. For instance, I can refer to "労働基準法（労基法）第89条" directly from the OCR without
  framing it as common knowledge. The user has specifically asked me to rely solely on the information provided.

  I’ll also check if there's a statement on "就業規則はどんな法律" in 2.md or another document since it might define work rules. Since 10.md focuses on purpose, I
  should search for "就業規則とは" to find relevant content.

• Explored
  └ Search 就業規則とは\|就業規則（\|規則という in モデル就業規則
    Read 2.md

• This is about setting standards for working conditions, which raises the question: "What laws govern employment rules?" We can mention the Labor Standards Act,
  particularly Article 89, which outlines the necessary elements of work rules. Also, there's the aspect of using company goods, referring to sections in the
  documentation like 16.md and 81.md. It's important to cite page numbers, possibly mentioning multiple pages since the format allows it, but I’ll verify what's
  needed.

  Determining citations and formats

  For Q1, I should use 10.md, page 10, which discusses the Labor Standards Act Article 89. For Q2, 16.md, page 16 includes necessary prohibitions, and I might
  consider mentioning 81 for additional context. While I could answer Q2 with 16 alone, adding 81 strengthens the response. I can include both page numbers since the
  formatting allows it. However, it might be clearer to focus on one primary source, citing both page 10 for purpose and page 2 for additional details if needed.

  Mapping to PDF and page
  Additionally, I should include that users can't use company facilities for personal reasons. It's crucial to mention the PDF as "モデル就業規則" and note it's the
  OCR file, ensuring to cite relevant excerpts and page numbers accordingly.

  I wonder if I should provide evidence for the page mapping, but it might not be necessary right now. It could be useful to check if MB20240318_2page has any
  relevant content regarding company goods. A quick search could clear that up. I want to ensure I'm referencing the right material, so I'll see what that page

  └ Search 物品\|備品\|施設\|私用\|職務以外 in MB20240318_2page
```

## claude code

回答

```
質問1: 就業規則はどんな法律？

回答

就業規則は、労働基準法（昭和22年法律第49号）
に基づく制度です。労働基準法第89条により、常時10人以上の労働者を使用する事業場では就業規則を作成し、所轄労働基準監督署長に届け出なければなりません。就業規則には、
労働時間・賃金・退職に関する事項など「絶対的必要記載事項」を必ず記載する必要があり、その内容は法令及び労働協約に反してはなりません（労基法第92条）。

どのPDFか

- モデル就業規則

どのページ数か

- 2ページ目、3ページ目
- 該当箇所の抜粋：
就業規則に記載する事項には、労働基準法（昭和22年法律第49号。以下「労基法」といいます。）第89条により、必ず記載しなければならない事項（以下「絶対的必要記載事項」
といいます。）と、各事業場内でルールを定める場合には記載しなければならない事項（以下「相対的必要記載事項」といいます。）があります。（2ページ目）

労基法は、労働者を1人でも使用する事業場に適用されますが、就業規則については、労基法第89条により、常時10人以上の労働者を使用する事業場においては、これを作成し又
変更する場合に、所轄労働基準監督署長に届け出なければなりません。（3ページ目）

なお、就業規則は、その内容が法令及び当該事業場において適用される労働協約に反してはなりません。法令又は労働協約に反する就業規則については、所轄労働基準監督署長は
その変更を命ずることができます（労基法第92条）。（3ページ目）

---
質問2: 会社の物品を会社とは関係ないところで使用していいの？

回答

いいえ、使用してはいけません。 就業規則の遵守事項として、「許可なく職務以外の目的で会社の施設、物品等を使用しないこと」と定められています。また、これに違反した場
は懲戒処分（懲戒解雇を含む）の対象となります。

どのPDFか

- モデル就業規則

どのページ数か

- 16ページ目（遵守事項の規定）、81ページ目（懲戒事由の規定）
- 該当箇所の抜粋：
第１１条 労働者は、以下の事項を守らなければならない。
1. 許可なく職務以外の目的で会社の施設、物品等を使用しないこと。（16ページ目）

⑩ 許可なく職務以外の目的で会社の施設、物品等を使用したとき。（81ページ目 ― 懲戒解雇の事由として記載）
```

実行中のエビデンス

```
● まず、files/ocr/ 配下にどのようなファイルがあるか確認します。
● Searched for 1 pattern (ctrl+o to expand)
● 「モデル就業規則」のファイルが多数あります。質問に関連しそうな箇所を検索します。
● Searched for 2 patterns (ctrl+o to expand)
● 関連するページを読み込みます。
● Read 5 files (ctrl+o to expand)
● 以下、files/ocr/ 配下の情報のみに基づいて回答します。
```

## claude code エージェントチーム

回答

```
質問1: 就業規則はどんな法律？
回答

就業規則は、労働基準法（昭和22年法律第49号） に基づく制度です。特に 労基法第89条 が就業規則の作成・届出義務の根拠となっており、常時10人以上の労働者を使用する事業
場では就業規則を作成し、所轄労働基準監督署長に届け出なければなりません。また、関連法として 労働契約法（平成19年法律第128号）
も就業規則と労働契約の関係を規定しています。

どのPDFか

モデル就業規則

どのページ数か

- 2ページ目
就業規則に記載する事項には、労働基準法（昭和22年法律第49号。以下「労基法」といいます。）第89条により、必ず記載しなければならない事項（以下「絶対的必要記載事項」
いいます。）と、各事業場内でルールを定める場合には記載しなければならない事項（以下「相対的必要記載事項」といいます。）があります。
- 3ページ目
労基法は、労働者を1人でも使用する事業場に適用されますが、就業規則については、労基法第89条により、常時10人以上の労働者を使用する事業場においては、これを作成し又は
変更する場合に、所轄労働基準監督署長に届け出なければなりません。
- 10ページ目
第1条 この就業規則（以下「規則」という。）は、労働基準法（以下「労基法」という。）第89条に基づき、　　　　株式会社の労働者の就業に関する事項を定めるものである。

---
質問2: 会社の物品を会社とは関係ないところで使用していいの？

回答

原則として禁止されています。モデル就業規則の第11条（遵守事項）において、「許可なく職務以外の目的で会社の施設、物品等を使用しないこと」と明記されています。違反し
場合は、軽度であれば「けん責・減給・出勤停止」、悪質な場合は「懲戒解雇」の対象となります。ただし「許可なく」という条件があるため、会社の許可を得れば使用が認めら
る余地はあります。

どのPDFか

モデル就業規則

どのページ数か

- 16ページ目（第11条 遵守事項）
第１１条 労働者は、以下の事項を守らなければならない。
1. 許可なく職務以外の目的で会社の施設、物品等を使用しないこと。
- 80ページ目（第64条第1項第5号 懲戒・軽度）
第64条 労働者が次のいずれかに該当するときは、情状に応じ、けん責、減給又は出勤停止とする。
⑤ 第11条、第12条、第13条、第14条、第15条に違反したとき。
- 81ページ目（第64条第2項第10号 懲戒解雇）
⑩ 許可なく職務以外の目的で会社の施設、物品等を使用したとき。
```

実行中のエビデンス

```bash
● まず、files/ocr/ 配下のファイル構成を確認し、2つの質問それぞれについて調査エージェントを並列で起動します。
● Searched for 1 pattern (ctrl+o to expand)
● ファイル構成を確認しました。「モデル就業規則」(89ページ)と「MB20240318_2page」(2ページ)があります。2つの質問について、調査エージェントを並列で起動します。
● 2 Task agents finished (ctrl+o to expand)
   ├─ 調査: 就業規則の法律 · 12 tool uses · 34.7k tokens
   │  ⎿  Done
   └─ 調査: 会社物品の私的使用 · 28 tool uses · 31.8k tokens
      ⎿  Done
● 両方の調査エージェントから結果が返ってきました。最終回答をまとめます。
```

## 考察

- 両方の答えについて正しく回答できている
- すべてのファイルを参照していというわけではない。
  - もう少しプロンプトの調査委が必要？
  - 漏れがないように書き出すとか？
- claude code エージェントチームは正しく動いているか判断が難しい
  - 「調査エージェントを並列で起動」と書かれているのでエージェントチームが動いていると考えていい？

## 注意点

- 今回はCLIでサクッと試しましたがアプリケーション位組み込む場合はそれぞれのSDKで実装する必要あり
  - claude code チームエージェントはまだ新しい機能なのでSDKがないかも。(chatgptはないといっている)
