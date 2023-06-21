---
title: "Apple IDでサインインできないからリジェクトされた話"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter"]
published: false
---

# はじめに
N-thy では Scade という音を録音しシェアしたり、マップ上に登録することができるアプリを開発しています．
https://apps.apple.com/jp/app/scade/id6448228456

今回はこのアプリを審査に出した際に「 Apple ID でサインインできない」という理由でリジェクトされた話についての記事です．

# 内容
Apple Store の審査に出すと、以下のような内容が返ってきてリジェクトされました．
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

内容を簡単にまとめると、「サードパーティーのログインサービスを使うなら Apple も入れないとリリースさせないぞ」ということです．
そのため現在開発中のアプリの Scade に Apple でのサインイン機能を追加しました．

# 前提
これまで firebase Auth を用いて Google アカウントでログインできるようになっていました．
また、マイページや Scade と呼ばれるユーザーに投稿された音声の一覧などでユーザーのプロフィール画像、表示名を取得して表示しています．

| ログイン画面 | マイページ | Scade |
| --- | --- | --- |
| ![](https://storage.googleapis.com/zenn-user-upload/eb04cd6d0ab3-20230525.jpeg) | ![](https://storage.googleapis.com/zenn-user-upload/f179ba5dc9a0-20230525.jpeg) | ![](https://storage.googleapis.com/zenn-user-upload/8164ed21ff54-20230525.jpeg) |

# 方針
こちらのパッケージを使用しました．
https://pub.dev/packages/sign_in_with_apple

# 実装
基本的には上記パッケージの README を参考に実装を行いました．
## BundleID の設定
:::message
今回はすでに基本的なアプリの設定が完了している前提なので、 BundleID の作成方法については紹介しません．
:::
以下のページから BundleID を選択します．
https://developer.apple.com/account/resources/identifiers/list/bundleId

そして、以下のようなページから SignInWithApple の Capabilities を選択して保存します．
![](https://storage.googleapis.com/zenn-user-upload/d16b4e8f535e-20230621.png)

## Firebase の設定
Firebase Authentication の Sign-in method で Apple を選択します．
以下のような画面で Apple を有効にします．
![](https://storage.googleapis.com/zenn-user-upload/755d4f69c48a-20230621.png)

## Xcode の設定
Xcode で `runner.xcodeprj` を開いて、 Capability から SignInWithApple を追加します．
![](https://storage.googleapis.com/zenn-user-upload/62a4e9b8da91-20230621.png)
![](https://storage.googleapis.com/zenn-user-upload/c45e7433fae6-20230621.png)

## 実装
```
$ flutter pub get sign_in_with_apple
```
などでパッケージをインストールします．
Signin のボタンを押すと、以下のようなロジックが呼ばれるようにします．
```dart
final credential = await SignInWithApple.getAppleIDCredential(
  scopes: [
    AppleIDAuthorizationScopes.email,
    AppleIDAuthorizationScopes.fullName,
  ],
);

final oauthProvider = OAuthProvider('apple.com');
final oAuthCredential = oauthProvider.credential(
  idToken: credential.identityToken,
  accessToken: credential.authorizationCode,
);

// 認証情報をFirebaseに登録
final user = (await FirebaseAuth.instance
        .signInWithCredential(oAuthCredential))
    .user;

// displayName, email が null なら空を入れる
if (user?.displayName == null ||
    (user?.displayName != null && user!.displayName!.isEmpty)) {
  final fixDisplayNameFromApple = [
    credential.givenName ?? '',
    credential.familyName ?? '',
  ].join(' ').trim();
  await user?.updateDisplayName(fixDisplayNameFromApple);
}
if (user?.email == null ||
    (user?.email != null && user!.email!.isEmpty)) {
  await user?.updateEmail(credential.email ?? '');
}
```

displayName, email の null チェックは、他の場所で displayName, email を使用する時のために行っています．
（ null になるケースがほとんどない & 特別な処理（ null だったら入力させるとか？）はコストが高い という理由から空を入れるという暫定的な対応をしています）

## その他
SignInWithApple ではユーザーのプロフィール画像を取得することができません．
そこでプロフィール画像が取得できなければデフォルトのアイコンを表示するようにするなどの変更も行いました．
今後はプロフィール画像などを後から変更できるようにもしていきたいと思っています．

