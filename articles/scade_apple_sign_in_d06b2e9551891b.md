---
title: "Apple IDでサインインできないからリジェクトされた話"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
N-thyでは Scade という音を録音しシェアしたり、マップ上に登録することができるアプリを開発しています．
https://apps.apple.com/jp/app/scade/id6448228456

今回はこのアプリを審査に出した際に「Apple IDでサインインできない」という理由でリジェクトされた話についての記事です．

# 内容
Apple Storeの審査に出すと、以下のような内容が返ってきてリジェクトされました．
```
Guideline 4.8 - Design - Sign in with Apple

Your app uses a third-party login service, but does not offer Sign in with Apple. Apps that use a third-party login service for account authentication need to offer Sign in with Apple to users as an equivalent option to provide the sign-in experience App Store users expect.

Next Steps

Please revise your app to offer Sign in with Apple as an equivalent option for account authentication.

Additionally, it would be appropriate to update the screenshots in your app's metadata to accurately reflect the revised app once Sign in with Apple has been implemented.

Resources

- Review Sign in with Apple sample code.
- For an overview of design and formatting recommendations for Sign in with Apple, see the Human Interface Guidelines.
- Learn about the benefits of Sign in with Apple.
```

内容を簡単にまとめると、「サードパーティーのログインサービスを使うならAppleも入れないとリリースさせないぞ」ということです．
そのため現在開発中のアプリの Scade にAppleでのサインイン機能を追加しました．


