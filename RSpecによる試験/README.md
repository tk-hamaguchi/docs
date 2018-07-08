RSpecによる試験
========

ユニットテストでの利用
--------

### Model Spec

Modelに対して必要なバリデータが設定されているか、必要なアトリビュートが意図した
権限で設定されているか確認するための試験。

メソッドを定義したりmoduleをincludeしている場合はそれらについても意図した動作を
しているか確認する（Concern含む）

【TBD】

また、各Modelに共通する関数定義についてはModelConcernへまとめて書き出すのが
望ましい。

### Controller Spec

Controllerの各アクションメソッド呼び出しに対して、インスタンス変数やレスポンス、flashが意図したものになっているか確認するための試験を記述する。

各アクションメソッドについてはパラメータの正誤とアクター毎にcontextを切って行う

```
RSpec.describe MyController, type: :controller do

  context '#update' do
    subject { put :update }
    let(:user_params) { double(ActionController::Parameters) }

    context 'with valid params' do
      before { allow(controller).to receive(:user_params).and_return(user_params) }
      context 'by confirmed_user' do
        let(:user) { FactoryBot.create :confirmed_user }
        let(:update_result) { true }
        before do
          allow(controller).to receive(:current_user).and_return(user)
          allow(user).to receive(:update).and_return(update_result)
          sign_in user
        end
        ### 中略 ###
      end
      context 'by unknown_user' do
        ### 中略 ###
      end
    end

    context 'with invalid params' do
      context 'by confirmed_user' do
        let(:user) { FactoryBot.create :confirmed_user }
        let(:update_result) { true }
        before do
          allow(controller).to receive(:user_params).and_raise(ActionController::ParameterMissing.new({}))
          sign_in user
        end
        ### 中略 ###
      end
    end

    context 'with invalid user_params' do
      context 'by confirmed_user' do
        let(:user) { FactoryBot.create :confirmed_user }
        let(:update_result) { false }
        before do
          allow(controller).to receive(:user_params).and_return(user_params)
          allow(controller).to receive(:current_user).and_return(user)
          allow(user).to receive(:update).and_return(update_result)
          sign_in user
        end
        ### 中略 ###
      end
    end

  end
```

subjectとしては各アクションに対して下記の３パターンの確認を記述し、加えてStrongParameterの処理等、アクションメソッド以外の確認についても記述する。

* レスポンスオブジェクト
* flashオブジェクト
* 各アクションが影響するインスタンス変数

記述する際には下記のポイントをスタブ化する

* アクションメソッドが呼び出している各メソッド
* モデルを含む外部モジュールのメソッド呼び出し



#### レスポンスオブジェクトの確認ポイント

* renderで終了するアクション（暗黙のrenderも含む）ではレスポンスステータスと、renderするview
* redirect_toで終了するアクションはリダイレクト先のパス

```
context '#top' do
  subject { get :top }
  context 'by confirmed_user' do
    context 'response' do
      subject { super(); response }
      before { sign_in FactoryBot.create :confirmed_user }
      it { is_expected.to have_http_status :ok }
    end
  end
  context 'by unknown user' do
    context 'response' do
      it { is_expected.to redirect_to new_user_session_path }
    end
  end
end
```

#### flashオブジェクトの確認ポイント

* alertやnoticeに含まれている文字列

```
context 'flash[:alert]' do
  subject { super(); flash[:alert] }
  it { is_expected.to eq I18n.t('devise.failure.unauthenticated') }
end
```

#### 各アクションが影響するインスタンス変数の確認ポイント


```
【TBD】
```

### StrongParameterの処理等、アクションメソッド以外についても意図した動作をしているか確認する（Concern含む）

```
context '#user_params' do
  let(:params) { {} }
  before do
    controller_params = ActionController::Parameters.new params
    allow(controller).to receive(:params).and_return(controller_params)
  end
  subject { controller.send(:user_params) }
  context 'params = {}' do
    it { expect{subject}.to raise_error ActionController::ParameterMissing }
  end
  context 'params = { user: { id: 1, name: "hoge" } }' do
    let(:params) { { user: { id: 1, name: 'hoge' } } }
    it { is_expected.to eq ActionController::Parameters.new({ name: 'hoge' }).permit! }
  end
  ### 中略 ###
end
```

確認ポイントは下記の通り

* requireに違反したパラメータをinputした時、`ActionController::ParameterMissing`がthrowされる
* permitしているものを全て指定した時、想定した`ActionController::Parameters`が返ってきている
* permitしているもの以外を含んだパラメータを指定した時、想定外のものが返り値から除外されている
* メソッド内で`#extract!`等を実行している場合、想定どおりに返り値から除外されている

また、各ControllerにはStrongParameterの確認メソッドと、アクションメソッド呼び出しの際に使用する小さなメソッドのみを記載する。viewからも利用するメソッドについてはHelperに、modelに関する操作はmodel（Concern含む）に書き、他の処理についてはControllerConcernにまとめて書き出すのが望ましい。


### View Spec

Viewについては各フォーマット毎、加えてレイアウトファイルやpartialファイルについても確認する。
確認ポイントはリンクやボタン等必要な要素が描画されているか、またパーツが意図した属性を付加された形で描画されているか、インスタンス変数やヘルパーの関数を使う場合はそれらを正しく利用できているか確認するための試験。

#### 必要な要素の描画

```
RSpec.describe 'users/sessions/new.html.haml', type: :view do

  subject { render ; rendered }

  it { is_expected.to have_title I18n.t('.title') }
 
  it { is_expected.to have_css '#login_form' }
  it { is_expected.to have_css :h1, text: I18n.t('app_name') }
  it { is_expected.to have_css :p, text: I18n.t('users.sessions.new.app_description') }

```

特にtitleタグのについては漏れがちなので、忘れずに各ページ毎に確認する

#### パーツが意図した属性を付加された形で描画されているか

```
  context 'with bootstrap' do
    context 'navbar' do
      it { is_expected.to have_css 'nav', class: 'navbar navbar-expand navbar-dark bg-dark fixed-top' }
      it { is_expected.to have_css 'nav > a', class: 'navbar-brand', text: I18n.t('app_name') }
      it { is_expected.to have_css 'nav > a.navbar-brand > img', class: 'd-inline-block align-top' }
      it { is_expected.to have_css 'nav > ul', class: 'navbar-nav ml-auto' }

```

#### partial ファイルが描画されているか

```
  before do
    stub_template 'layouts/_alerts.html.haml' => 'ALERTS_CONTENTS'
  end

  # render partial: '/layouts/alerts'
  it { is_expected.to have_css '#login_form', text: 'ALERTS_CONTENTS' }
```

#### layout内に複数のyieldが存在している場合

header等に要素を埋め込む記述がある場合はそれも確認する

```
require 'rails_helper'
  
RSpec.describe 'layouts/my.html.haml', type: :view do
  let(:grid_system_layout) {
    <<~"EOT"
      !!!
      %html
        %head
          %title
            GRID_SYSTEM_LAYOUT
          = yield :stylesheet_links
          = yield :javascript_links
        %body
          = content_for?(:application_content) ? yield(:application_content) : yield
    EOT
  }
  before do
    stub_template 'layouts/grid_system.html.haml' => grid_system_layout
  end
  subject { render; rendered }

  ### 中略 ###

  it { is_expected.to have_css 'head link[rel="stylesheet"][media="screen"][href^="/packs-test/my-"]', visible: false }
  it { is_expected.to have_css 'head script[src^="/packs-test/my-"]', visible: false }

end
```

#### インスタンス変数やヘルパーの関数

```
TBD
```

### Routing Spec

HTTPメソッドとリクエストパスが意図したコントローラー・アクションへ落ちているか確認するための試験。

ファイル化する単位はルーティングを指定する行単位で行う
```
Rails.application.routes.draw do
  get '/version' => 'api#version', defaults: { format: :text }   #=> (1)
  namespace :my do
    resources :todos   #=> (2)
  end
  root to: 'home#index'  #=> (3)
end
```

| 脚注番号 |                  ファイル                  |
|:--------:|:-------------------------------------------|
|    (1)   | `spec/routing/api_version_routing_spec.rb` |
|    (2)   | `spec/routing/my_todos_routing_spec.rb`    |
|    (3)   | `spec/routing/root_to_routing_spec.rb`     |

subjectとしては下記の２パターンを記述する

* HTTPメソッドとリクエストパス、クエリパラメータを指定する形
* `*_path`メソッドからの逆引き

確認ポイントとしては下記の２パターンを記述する

* `routable`である
* コントローラやアクション、ある場合はパラメータやデフォルト値について意図した
形で解析されている

```
RSpec.describe "get '/version', defaults: { format: :text }", type: :routing do
  context 'GET /version' do
    subject { get '/version' }
    it { is_expected.to be_routable }
    it { is_expected.to route_to controller: 'api', action: 'version', format: :text }
  end

  context 'GET /version?details=true' do
    subject { get '/version?details=true' }
    it { is_expected.to be_routable }
    it { is_expected.to route_to controller: 'api', action: 'version', format: :text, details: 'true' }
  end

  context 'version_path' do
    subject { get version_path }
    it { is_expected.to be_routable }
    it { is_expected.to route_to controller: 'api', action: 'version', format: :text }
  end
end
```

アクターまたはロール毎に振る舞いが異なる場合は、必要に応じて各アクター・ロール毎の試験パターンを記述する


### Helper Spec

【TBD】


### Job Spec

【TBD】


### Mail Spec

【TBD】



インテグレーションテストでの利用
--------

### Request Spec

HTTPメソッドとリクエストパス、パラメータ、ボディをinputとして、意図したviewを描画しているか、ステータスコードは意図したものになっているか確認するための試験。

原則としてHTMLファイルを組み立てて描画する単体のリクエストをsubjectとする。
またsubjectの指定はHTTPメソッドとリクエストパス、クエリパラメータを指定する形で行う。
index等の事前にデータを登録する必要があるものについてはデータ登録をbeforeブロックで行う

```
subject { get '/my' }
```

確認ポイントとしては下記の通り

* partialも含め、意図したlayoutファイルが描画されているか
* layoutファイルに条件分岐が含まれている場合は描画されていないこと

```
RSpec.describe 'GET /my', type: :request do
  subject { get '/my' }
  context 'with unknown user' do
    it { is_expected.to redirect_to new_user_session_path }
  end

  context 'with signed in user' do
    before { sign_in FactoryBot.create :confirmed_user }

    it { is_expected.to render_template :application }
    it { is_expected.to render_template partial: 'layouts/_header' }

    it { is_expected.to render_template :grid_system }
    it { is_expected.to render_template partial: 'layouts/_side_menu_items' }
    it { is_expected.to render_template partial: 'layouts/_footer' }

    it { is_expected.to render_template :my }
    it { is_expected.to render_template partial: 'layouts/_alerts' }

    it { is_expected.to render_template 'my/top' }

    context 'response' do
      subject { get '/my'; response }
      it { is_expected.to have_http_status :ok }
    end
  end
end
```


### API Spec

仕様に規定されている各WebAPI機能に対し、シナリオレベルでの動作確認を行う。
前提条件が必要な場合は、その前提も含む形でfeatureとscenarioを記述する

機能要件の１項目を１シナリオとして記述する

```
RSpec.feature 'バージョン確認API', type: :request do
  context 'GET /version' do
    context 'without params' do
      scenario {
        get '/version'
        expect(response).to have_http_status :ok
        expect(response).to render_template 'api/version'
        expect(response.body).to match /APP_VERSION: #{Rails520v::VERSION}$/
        expect(response.body).to_not match /DB_VERSION:/
        expect(response).to be_successful
      }
    end

    context 'with details=1' do
      scenario {
        get '/version?details=1'
        expect(response).to have_http_status :ok
        expect(response).to render_template 'api/version'
        expect(response.body).to match /APP_VERSION: #{Rails520v::VERSION}$/
        expect(response.body).to match /DB_VERSION:  \d+$/
        expect(response).to be_successful
      }
    end
  end
end
```

### System Spec

仕様に規定されている各WebUIベースの機能に対し、シナリオレベルでの動作確認を行う。
前提条件が必要な場合は、その前提も含む形でfeatureとscenarioを記述する

機能要件の１項目を１シナリオとして記述する


```
RSpec.feature 'アカウント情報変更機能', type: :system do
  let(:tenant) { FactoryBot.create :tenant }
  let(:user)   { FactoryBot.create :confirmed_user, tenant_id: tenant.id }

  context '登録済ユーザは' do
    context 'アカウント設定ページから' do

      scenario 'アカウント情報を変更することができる', js: :true do
        # ホームページを表示
        visit root_path

        # ログインリンクをクリック
        find(:css, "nav a[href=\"#{new_user_session_path}\"]").click

        # ログインフォームに入力
        within(:css, "form[action=\"#{user_session_path}\"][method=\"post\"]") do
          fill_in 'メールアドレス', with: user.email
          fill_in 'パスワード',     with: user.password
          click_button 'ログイン'
        end

        expect(page.current_path).to eq '/my'
        expect(page).to have_css :div, text: 'ログインしました。'

        # ドロップダウンからアカウント設定リンクをクリック
        click_link user.name
        click_link 'アカウント設定'

        expect(page.current_path).to eq '/my/edit'

        # アカウント設定ページのアカウント設定フォームに入力
        within(:css, "form#edit_user_#{user.id}") do
          fill_in 'ユーザ名', with: 'ほげ'
          fill_in 'パスワード',           with: 'psWd123'
          fill_in 'パスワード（確認用）', with: 'psWd123'
          click_button 'アカウント設定を更新'
        end

        expect(page.current_path).to eq '/users/sign_in'
        expect(page).to have_css :div, text: 'アカウント登録もしくはログインしてください。'

        # 新しいパスワードでログインしてみる
        within(:css, "form[action=\"#{user_session_path}\"][method=\"post\"]") do
          fill_in 'メールアドレス', with: user.email
          fill_in 'パスワード',     with: 'psWd123'
          click_button 'ログイン'
        end

        # アカウント設定フォームに含まれているユーザ名が更新されている
        expect(page).to have_css :div, text: 'ログインしました。'
        expect(page).to have_link 'ほげ'
        within(:css, "form#edit_user_#{user.id}") do
          expect(page).to have_field 'user[name]', with: 'ほげ', id: 'user_name'
        end
      end
    end
  end
end
```
