---
layout: post
title: "Flutterでアプリ作ってみたPt3"
date: 2024-12-16 00:00:00 +0900
categories: car
---

この記事は、[貴族会 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/kizokukai) 16 日目の記事です。

[昨日](https://5hal1n.github.io/blog/car/2024/12/15/create-flutter-app-pt2.html)の続きです。

元々書こうとしていた内容を思い出せず、急いでアプリ作りながら記事も書いてます。

## 何をするか

今回は、ログインしていない時はログインページを表示し、GoogleSignIn する、までをやってみようと思います。

## CLI ゴニョゴニョ

```bash
flutter pub add firebase_auth
flutter pub add google_sign_in
flutter pub add sign_in_button
flutterfire configure # yesで進める
```

## ファイルゴニョゴニョ

### ~/ios/Podfile

前回まではどんなデバイスでも動きましたが、今回は iOS をターゲットに実装
そしてちょっと Podfile をいじる必要がありました。

```plain
platform :ios, '18.0'
```

バージョンを幾つにするかは、ターゲットユーザによりますが、今回はわかりやすく 18 を指定

その後ターミナルで、`pod repo update`を実行しておきましょう。

### ~/ios/Runner/info.plist

info.plist に、以下のコードを適切いい感じに設定しましょう。

```plist
<key>CFBundleURLTypes</key>
<array>
  <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLSchemes</key>
        <array>
          // GoogleService-info.plistのREVERSED_CLIENT_IDに置き換える
          <string>REVERSED_CLIENT_ID</string>
        </array>
  </dict>
</array>
```

### ~/lib/main.dart

main.dart を書き換えていきましょう。

markdown でいい感じに差分を出す方法を知らないので、今回もコードベタ張りです。

許してね

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:google_sign_in/google_sign_in.dart';

import 'package:firebase_core/firebase_core.dart';
import 'package:sign_in_button/sign_in_button.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (context) => MyAppState(),
      child: MaterialApp(
        title: 'Namer App',
        theme: ThemeData(
          useMaterial3: true,
          colorScheme: ColorScheme.fromSeed(seedColor: Colors.blueAccent),
        ),
        home: MyHomePage(),
      ),
    );
  }
}

class MyAppState extends ChangeNotifier {
  final myController = TextEditingController();
  List<Map<String, dynamic>> _logs = [];
  List<Map<String, dynamic>> get logs => _logs;
  Stream<QuerySnapshot> get logsStream =>
      FirebaseFirestore.instance.collection('logs').snapshots();

  final auth = FirebaseAuth.instance;
  User? user;

  Future<void> signInWithGoogle() async {
    final googleUser = await GoogleSignIn().signIn();
    if (googleUser == null) {
      return;
    }

    final googleAuth = await googleUser.authentication;
    GoogleAuthProvider provider = GoogleAuthProvider();
    provider.addScope('https://www.googleapis.com/auth/contacts.readonly');

    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );

    final userCredential = await auth.signInWithCredential(credential);
    user = userCredential.user;
    notifyListeners();
  }

  Future<void> signOut() async {
    await auth.signOut();
    user = null;
    notifyListeners();
  }

  Future<void> _fetchLogs() async {
    try {
      final querySnapshot =
          await FirebaseFirestore.instance.collection('logs').get();
      _logs = querySnapshot.docs.map((doc) => doc.data()).toList();
      notifyListeners();
    } catch (e) {
      print('Error fetching logs: $e');
    }
  }

  void save() {
    FirebaseFirestore.instance.collection('logs').add({
      'title': myController.text,
      'timestamp': DateTime.now(),
    });
    myController.clear();
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key});
  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  late MyAppState _appState;
  var selectedIndex = 0;
  final myController = TextEditingController();

  @override
  void dispose() {
    myController.dispose();
    super.dispose();
  }

  @override
  void initState() {
    super.initState();
    _appState = MyAppState(); // インスタンス化
    _appState._fetchLogs(); // 初期化時にデータを取得
  }

  @override
  Widget build(BuildContext context) {
    Widget page;
    if (_appState.user == null) {
      page = Center(
          child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          SignInButton(
            Buttons.google,
            onPressed: () async {
              await _appState.signInWithGoogle();
            },
          )
        ],
      ));
    } else {
      switch (selectedIndex) {
        case 0:
          page = GeneratorPage();
          break;
        case 1:
          page = ListPage();
          break;
        case 2:
          page = LogoutPage();
          break;
        default:
          throw UnimplementedError('no widget for $selectedIndex');
      }
    }

    return LayoutBuilder(builder: (context, constraints) {
      return Scaffold(
        body: Row(
          children: [
            SafeArea(
              child: NavigationRail(
                extended: constraints.maxWidth > 600,
                destinations: [
                  NavigationRailDestination(
                    icon: Icon(Icons.home),
                    label: Text('Home'),
                  ),
                  NavigationRailDestination(
                    icon: Icon(Icons.favorite),
                    label: Text('favorite'),
                  ),
                  NavigationRailDestination(
                    icon: Icon(Icons.logout),
                    label: Text('Logout'),
                  ),
                ],
                selectedIndex: selectedIndex,
                onDestinationSelected: (value) {
                  setState(() {
                    selectedIndex = value;
                  });
                },
              ),
            ),
            Expanded(
              child: Container(
                color: Theme.of(context).colorScheme.primaryContainer,
                child: page,
              ),
            ),
          ],
        ),
      );
    });
  }
}

class GeneratorPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var appState = context.watch<MyAppState>();

    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          TextField(
            controller: appState.myController,
            decoration: InputDecoration(
              hintText: 'Input free text',
            ),
          ),
          SizedBox(height: 10),
          Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              ElevatedButton.icon(
                onPressed: () {
                  appState.save();
                },
                icon: Icon(Icons.save),
                label: Text('Save'),
              ),
            ],
          ),
        ],
      ),
    );
  }
}

class ListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var appState = context.watch<MyAppState>();

    return StreamBuilder<QuerySnapshot>(
      stream: appState.logsStream,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Text('Error: ${snapshot.error}');
        }

        if (snapshot.connectionState == ConnectionState.waiting) {
          return Text('Loading...');
        }

        return ListView(
          children: snapshot.data!.docs.map((DocumentSnapshot document) {
            Map<String, dynamic> data =
                document.data()! as Map<String, dynamic>;
            return ListTile(
              title: Text(data['title'] ?? ''),
            );
          }).toList(),
        );
      },
    );
  }
}

class LogoutPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var appState = context.watch<MyAppState>();

    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Text('Logout'),
          ElevatedButton(
            onPressed: () {
              appState.auth.signOut();
            },
            child: Text('Logout'),
          ),
        ],
      ),
    );
  }
}
```

これで一通り、GoogleSignIn して、画面が（タブを選択すると切り替わり）、サインアウトできる（Hot restart）すると反映される

## 次回

ここまでは CodeLab の UI をそのまま使い回してきましたが、SNS っぽい見た目のアプリを作りたいので、少し UI をいじったり、SignIn/SignOu が即時に UI に反映されるようにしていきます。
