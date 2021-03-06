# rubocopによるチェック

Rubyは１つのことを実行するにも様々な書き方ができることが特徴の言語です。これは学習コストを下げ、併せて参入障壁を下げる効果もありますが、複数名でソースコードを共有してプロダクトを開発する場合はこのメリットがデメリットに変わるケースがあります。各々が思い思いの書き方をした場合、レビューを実施する側から見たときに可読性が下がってしまいます。これにより、コードレビューやプルリク承認時のコード確認に時間がかかったり見落としが起きることがあります。

それを防ぐために一定のコーディング規約を定め、それを遵守することが、全体的な効率化や品質向上につながります。

・・・とは言っても、毎回全てのコードを目で見て、人手でチェックすることは非現実的なため、昨今ではツールを使ってコーディング規約の遵守状況のチェックをします。

今回はそのツールの１つであるrubocopの設定についていくつかのパターンを下記に記します。


## 使う上での注意

rubocopはコードを静的に評価しますが、チームのレベルや実装方式によっては遵守が難しいケースが出てきます。設定ファイルの更新によるルール変更や、一部のソースコードに対するコメントを利用した許可を出す場合、チーム内での合意形成が必要になります。この合意形成ルールについても、予め決めておいたほうがスムーズに開発が進みます。

### 例：リポジトリオーナーが責任を持っているケース

1. ログメッセージの出力は一行の長さに対するルールは適用しない
2. コメントによる無効化・設定ファイルの変更はプルリクの要求者がプルリク内で妥当性・必要性を説明し、リポジトリオーナーが必要性を判断する
3. 設定ファイルやルールに対して変更が発生した場合はリポジトリオーナーがチームに対してアナウンスを行う

**リポジトリオーナーとは：** 筆者のプロジェクトの場合、rails newしたものをプッシュし、プルリクのマージや承認を行う役割をリポジトリオーナーとして設定しています。大きめのプロジェクトの場合や、チームメンバーのスキルにばらつきがある場合は別途レビュアーを設けますが、マージの承認は必ずリポジトリオーナーが判断して行います。

### 例：小さなチーム(5人以下)でリポジトリオーナーを決めないケース

1. ログメッセージの出力は一行の長さに対するルールは適用しない
2. ルールや設定ファイルの変更はチーム内での投票を持って決定する
3. チームのスキルレベルや責任範囲に差がある場合、スキルレベルや責任範囲に応じた投票ポイントを追加で付与する
4. チーム内の投票ポイントの8割の賛成票で変更を実施する
5. チームの8割が揃わないときは投票を行わない


## 対象バージョン

この設定はRuby2.5.x系、Rubocop0.60.x系に最適化されています。

## Railsプロジェクト用のルール

* 辛口： RailsやGemのジェネレータが生成したファイルも含めてコミットする人間が全て修正を行う
* 中辛： RailsやGemのジェネレータが生成したファイルは必要最低限のチェックを、他のファイルには必要なチェックを行う
* 甘口： メソッド分割やモジュール分割のスキルが未成熟なチーム用にABCサイズを調整したり、テストコードやGemfileに対するチェックをスルーするように調整したもの。一見難易度は下がっているように見えるが、中辛以上のファイルを適用した場合と比較してユニットテストの難易度が上がり、結果技術的負債を貯めがちになる点に注意する必要がある

## RubyGemsプロジェクト用のルール
