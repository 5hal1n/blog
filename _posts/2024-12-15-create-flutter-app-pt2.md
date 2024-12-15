---
layout: post
title: "Flutterでアプリ作ってみたPt2"
date: 2024-12-15 00:00:00 +0900
categories: car
---

この記事は、[貴族会 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/kizokukai) 15 日目の記事です。

[前回](https://5hal1n.github.io/blog/car/2024/12/11/create-flutter-app-pt1.html)の続きです。

## 何をするか

今回は、TextField に入力したものを Firestore に保存して、それをリアルタイムで表示するところまでやってみます。

## CLI ゴニョゴニョ

とりあえず以下は用意されている前提で、進めていきます。

- Firebase のプロジェクト
- Firebase の[CLI](https://firebase.google.com/docs/cli?hl=ja#setup_update_cli)

以下ターミナルで操作

```bash
firebase login
dart pub global activate flutterfire_cli

exec $SHELL -l

flutterfire configure # 対話形式で進むのでよしなに進める

flutter pub add firebase_core
flutter pub add cloud_firestore
flutterfire configure # yesで進める
```

## ファイルゴニョゴニョ

なんかファイルが生成されたりパッケージが更新されてたりすると思います、進めていきます

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import 'package:firebase_core/firebase_core.dart';
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
    _appState = MyAppState();
    _appState._fetchLogs();
  }

  @override
  Widget build(BuildContext context) {
    Widget page;
    switch (selectedIndex) {
      case 0:
        page = GeneratorPage();
        break;
      case 1:
        page = ListPage();
        break;
      default:
        throw UnimplementedError('no widget for $selectedIndex');
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
```

Pt1 との差分があればいいのですが、前回終了時に commit してなかったり、今回も作り終えてからあー疲れたって commit しているので、差分がないです。

そして Gemini や Cursor を駆使してなんとか動かしているので、全てを理解してるわけでもありません。

むしろ理解してないことの方が多い。

でも動けばいい

ぜひローカルの main.dart にコピペして、flutter run して動かしてみてください。
よくわからんアプリが出来上がります。

## 次回

ダラダラと flutter アプリを作るの意外と楽しいのでまだまだ擦ります。

次回は認証の機構を用意しようと思います。
