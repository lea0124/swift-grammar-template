# 読解ノート：自分のアプリのコード3箇所

> **氏名：はやむ　、　ムアタル
> **AI利用レベル：2（生成コード可）
> **最終更新：** 2026-11-09

生成AIに書いてもらったコードでも、自分で書いたコードでもかまいません。自分のアプリの中から3箇所を選んで、次の3つの問いに答えてください。分からないところは、生成AIに「この行は何をしていますか」「この行を消すとどうなりますか」と聞いてかまいません。ただし、書くのは自分の言葉です。

> 💡 担任の先生の指示で、項目が増えることがあります。

---

## 箇所1：「ゲームスタート」を押したときの処理（tapStartGame・startGame　3976〜3997行）

《コード》
    private func tapStartGame() {
        if isBlessingName(playerName) {
            showBlessing()
        } else {
            startGame(blessed: false)
        }
    }

    private func startGame(blessed: Bool) {
        DropSoundPlayer.shared.preload(Array(Set(ballStyles.indices.map { soundName(for: $0) })))
        MergeSoundPlayer.shared.prepare()
        restart()
        // （blessed のときは 5000円スタート・アイテム1回無料などを追加：3989〜3995行）
        isShowingGame = true
    }

何をしているか：
「ゲームスタート」を押すと、まず入力した名前が特別な名前かどうかを調べる。特別な名前なら「原初者の祝福」の画面を出し、ふつうの名前ならそのままゲームを始める。始めるときは、効果音を先に読みこみ、restart() でゲームを最初の状態にもどしてから、isShowingGame を true にしてゲーム画面に切りかえる。

なぜこう書くのか：
・画面が「ホーム」と「ゲーム」の2つだけなので、body（1204〜1215行）では isShowingGame という Bool 1つを if 〜 else で見て、どちらを出すか決めている。画面遷移のしくみを使うより簡単。
・preload は、最初に猫を落としたときに音がおくれないように、先に音声ファイルを読みこんでおくため。Set にしているのは、犬のボールのように同じ音を使うボールがあるので、重複を消すため。
・スタートのたびに restart() を呼ぶので、前のゲームのボールやお金が残らない。
・祝福のあるときとないときで、同じ startGame を blessed という引数で使い分けているので、ゲームを始める処理は1か所ですむ。

もしこう書かなかったら：
・変えるところ：3987行の restart() を消す。
・予想：1回目はたぶんふつうに始まる。でも、ゲームオーバーになって「ホームに戻る」を押し、もう一度「ゲームスタート」を押すと、gameOver が true のままなので、いきなり結果画面が出ると思う。途中で終了したときは、前のボール・お金・タイムが残ったまま続きから始まるはず。
・実際に試した結果：（試したらここに書く）

---

## 箇所2：リスタートの確認と、状態のリセット（requestRestart 3893〜3899行、restart 3921〜3925行）

《コード》
    private func requestRestart() {
        if gameOver || isVictory || (dropCount == 0 && fruits.isEmpty) {
            restart()
        } else {
            pendingAction = .restart
        }
    }

    private func restart() {
        DropSoundPlayer.shared.stop()
        AdSoundPlayer.shared.stop()
        fruits.removeAll()
        money = 0
        // …（このあと 3973行まで、タイム・コンボ・アイテムの値段など約50個の値を元にもどす）
    }

何をしているか：
リスタートボタンを押したとき、ゲームがもう終わっているか、まだ何もしていないなら、確認なしですぐ restart() する。ゲームの途中なら pendingAction に .restart を入れて、「リスタートしますか？」の確認を出す。確認で「リスタート」を選ぶと perform（3909行）で途中までの記録を「未クリア」として保存してから restart() を呼ぶ。restart() は、鳴っている音を止めて、ボール・お金・タイムなど、ゲームの状態を全部最初にもどす。

なぜこう書くのか：
・ゲームの途中で押しまちがえると、それまでのプレイが消えてしまうので確認を出している。終わったあとなら消えて困るものがないので、確認はいらない。
・pendingAction に値が入っているあいだは一時停止（isPaused）になるので、確認が出ているあいだはタイムもボールも止まる。
・「リスタート」と「終了」は確認の出し方が同じなので、PendingAction という enum で1つの確認にまとめている。
・もどす値がとても多い。機能をふやすたびに状態がふえたので、1つでも書きわすれると、前のゲームの状態が次のゲームに残るバグになる。だから、もどす処理を restart() 1か所に集めて、ゲームスタートのときもこれを使っている。

もしこう書かなかったら：
・変えるところ：3925行の money = 0 を消す。
・予想：リスタートしても、前のゲームで持っていたお金がそのまま残る。最初からアイテムが買えるので、前より簡単になってしまう。ホームに戻ってからスタートしても、startGame の中で restart() を使っているので同じことが起きると思う。（祝福の名前で始めたときだけは、お金が5000円に上書きされる。）
・実際に試した結果：（試したらここに書く）

---

## 箇所3:テストで直したタッチ操作（gameBoard の DragGesture・1748〜1763行）

《コード》
    .gesture(
        DragGesture(minimumDistance: 0)
            .onChanged { value in
                guard !gameOver, !isVictory, !isPaused, eggPhase == nil, fallingDogID == nil, isReadyToDrop, !isWindBlowing else { return }
                moveHeldFruit(to: value.location.x - Game.sideWallInset)
            }
            .onEnded { value in
                guard !gameOver, !isVictory, !isPaused, eggPhase == nil, fallingDogID == nil, isReadyToDrop, !isWindBlowing else { return }
                moveHeldFruit(to: value.location.x - Game.sideWallInset)
                dropFruit()
            },
        // ゲームオーバー中は「ボールを落とす操作」だけを止める（.subviews）
        // ※ .none にすると中のボタンまで押せなくなる（iOS 16 で発生）
        including: gameOver ? .subviews : .all
    )

何をしているか：
箱の上で指を動かすと、持っている猫が左右に動き、指をはなすとその場所から落ちる。minimumDistance: 0 なので、動かさずにタップしただけでも落とせる。guard で、ゲームオーバー中・一時停止中・ルーレット中・犬が落ちているあいだ・続けて落とせないあいだなどは、何もしないようにしている。最後の including で、ゲームオーバー中はこのドラッグ操作だけを止め、画面の中のボタンは押せるようにしている。

なぜこう書くのか：
・.none にすると、iOS 16 では中のボタン（広告で復活・もう一度遊ぶ・ホームに戻る）まで押せなくなる不具合があった。.subviews にすると、このドラッグだけが止まって、中のボタンは生きたままになる。
・iOS 17 以降では .none でもボタンが動いてしまうので、新しい iPhone だけでテストしていると気づけない。いろいろな iOS のバージョンの実機で確かめることが大事だと分かった。
・guard の条件が多いのは、機能を足すたびに「このあいだは落とせてはいけない」場面がふえたから。たとえば fallingDogID（ルーレットの犬が落ちているあいだ）は、速くタップすると犬の効果が出る前に次の猫が落ちてしまう不具合があったので、条件に足した。
・onChanged と onEnded の両方に同じ条件を書いているのは、指を動かしている途中で状態が変わっても（ゲームオーバーになった、など）、はなしたときにもう一度確かめるため。

もしこう書かなかったら：
・変えるところ：1762行の .subviews を .none に変える。
・予想：iOS 16 の iPhone では、ゲームオーバー画面のボタンが全部押せなくなって、ホームにも戻れなくなる。iOS 17 以降の iPhone では、たぶんふつうに押せてしまう（だからこの不具合は見つけにくかった）。
・実際に試した結果：（試したらここに書く）

---

## 自分ではまだ説明できない箇所（正直に）

・Timer.publish(every:on:in:).autoconnect() で作ったタイマーと、onReceive(gameTimer)（1516行）のしくみ。これで1秒に60回ゲームが進むのは分かるが、なぜそうなるのかは分からない。
・scheduleBlessingCheck（3195〜3207行）で、UUID の token をくらべて「古い予約」を無視しているところ。なぜ UUID を使うと古いほうだけが無視されるのか、まだうまく説明できない。
・キーボードを閉じる UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), ...)（3213行）。#selector が何なのか分からない。
