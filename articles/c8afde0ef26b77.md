---
title: "PythonでMisskeyのBotを作る"
emoji: "⛳"
type: "tech"
topics:
  - "python"
  - "discordpy"
  - "framework"
  - "misskey"
published: true
published_at: "2023-07-05 23:55"
---

# はじめに

最近何かと話題のMisskeyですが、今回は有名なMisskey.pyではなく、MiPAというフレームワークを使ってBotを作ってみます。

## MiPAとは

MiPAは私自身が作成しているMisskey用のBotフレームワークで、特徴としてはDiscord.pyのコードを参考に作成しているため、Cogやイベント等の記述方法に類似点が多いため、Discord.pyの知識をそのまま活かせるところです。また、独自の機能のとして、 `@ユーザー名 コマンド` のようなコマンドを作成できるメンションコマンドというシステムもあります。今回はこれらの機能を使ってBotを作っていきます。

:::details MiPAとMisskey.pyの違いについて
Misskey.pyはBotフレームワークではなく、あくまでAPIラッパーです。また、戻り値などは型が無く、ほとんど生のdictでレスポンスが帰ってきます。それに対し、MiPAではDiscord.pyのようにそれぞれのモデルに応じてクラスが存在するため、ノートでは `Note.author.id` のようにプロパティで取得でき、また時間なども`str`から`datetime`に自動で変換します。何より型がはっきりしています。

また、Misskey.pyは前述したとおりAPIラッパーなのでWebsocket等を持ちません。Misskey.pyでBotを作成する場合、`websockets` 等のライブラリを別に入れ、自分でセッションの確立、イベントの管理等をしないといけません。これらは個人的にBotを作る側としてはあまり関心がないことだと思います。MiPAはDiscord.pyと同様にWebsocketの接続を持ち、イベント等もDiscord.pyと類似した書き方で書くことができるため、不必要な知識は要りません。

より詳細な変更点については以下の表をご覧ください。
ここでは公平に機能面で違いを図るため、APIラッパーとしての機能だけを記述します。

|項目|misskey.py|MiPA|
|---|---|---|
|戻り値に型があるか|✖|〇|
|トークンの生成をサポートしているか|〇|〇|
|ドキュメントが充実しているか|〇|△|
|エンドポイントがすべてサポートされているか|△|△|
|最新のエンドポイントと古いエンドポイントの互換性があるか|△|〇|
|テストがあるか|〇|✖|
|LICENSE|MIT|MIT|
|必要なPythonのバージョン|Python3.7以上|Python3.11以上|
:::

## インストール

```bash
pip install mipa
```

## 実際にBotを作る

以下のコードで ホームタイムラインと自身の通知などを受け取るBotが作成できます。
グローバルタイムライン等に接続したい場合は `self.router.connect_channel` メソッドの配列に渡すものに `global` を追加すればいけます。

```python:main.py
import asyncio

from aiohttp import ClientWebSocketResponse
from mipac import NoteDeleted, PartialReaction, NoteReaction, Note

from mipa.ext.commands.bot import Bot


class MyBot(Bot):
    def __init__(self):
        super().__init__()

    async def on_ready(self, ws: ClientWebSocketResponse):
        await self.router.connect_channel(['main', 'home'])
        print(self.user.username)

    async def on_note(self, message: Note):
        print(message.author.name, message.content)

if __name__ == '__main__':
    bot = MyBot()
    asyncio.run(bot.start('server url', 'token'))
```

## ノートを投稿してみる

ここらへんはかなりDiscord.pyとは異なります。
今回は先ほどの作成したコードに追加する形で作成してみましょう。

```diff
    async def on_ready(self, ws: ClientWebSocketResponse):
        await self.router.connect_channel(['main', 'home'])
        print(self.user.username)
+        await self.client.note.action.send('Botが起動しました')
```

Discord.pyでは、`Client.fetch_channel` のようなものが、MiPAではそれぞれの分野に分けられており、 ユーザーを取得する場合は `self.client.user.action.get('ユーザーのID')` といった形のような形で取得が可能です。

## Cogを作る

今回は試しに `@botの名前 hello`と打つと `こんにちは！○○　さん`と返すコマンドを作ってみます。
また、以下のようなディレクトリ構造で作成していきます。

```
.
├── cogs
│   └── basic.py
└── main.py
```

```python:main.py
import asyncio

from aiohttp import ClientWebSocketResponse
from mipac import Note

from mipa.ext.commands.bot import Bot

COGS = ["cogs.basic"]

class MyBot(Bot):
    def __init__(self):
        super().__init__()

    async def on_ready(self, ws: ClientWebSocketResponse):
        for cog in COGS:
            self.load_extension(cog)
        await self.router.connect_channel(['main', 'home'])
        print(self.user.username)

    async def on_note(self, message: Note):
        print(message.author.username, message.content)


if __name__ == '__main__':
    bot = MyBot()
    asyncio.run(bot.start('url', 'token'))
```

```python:basic.py
class BasicCog(commands.Cog):
    def __init__(self, bot: Bot) -> None:
        self.bot = bot
    
    @commands.mention_command(text='hello')
    async def hello(self, ctx: Context):
        await ctx.message.api.action.reply(f'こんにちは！ {ctx.message.author.username} さん')

def setup(bot: Bot):
    bot.add_cog(BasicCog(bot))
```

### 正規表現を用いてタイマーコマンドを作ってみる

メンションコマンドは `regex` と `text` のどちらかを使用でき、regexでは正規表現のパターンを渡すことで、それに一致するものを位置引数で返します。
今回は先ほど作成した`basic.py`を編集する形で作成します。

```python
class BasicCog(commands.Cog):
    ...

    @commands.mention_command(regex=r'(\d+)秒タイマー')
    async def timer(self, ctx: Context, time: str):
        await ctx.message.api.action.reply(f'{time}秒ですね。よーいドン！')
        await asyncio.sleep(int(time))
        await ctx.message.api.action.reply(f'{time}秒経ちました！')
```

### 最後に

今回は私が作成しているMiPAというライブラリを使ってMisskeyのBotを作ってみました。実はMiPAは内部でMiPACというライブラリを使っており、これがAPIラッパーとしての機能を提供しています。興味がある方は、最後に記載しているリンク集から飛んでみてみてください。
また、何か分からないことがあれば私のMisskeyアカウント（@yupix@nr.akarinext.org） に対してメンションを飛ばしたりすることでいつでも聞けるので、気軽にご相談ください！

### リンク集

https://github.com/yupix/MiPA
https://github.com/yupix/MiPAC
https://github.com/YuzuRyo61/Misskey.py
