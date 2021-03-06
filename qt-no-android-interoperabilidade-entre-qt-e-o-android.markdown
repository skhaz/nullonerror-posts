
<p>{% imgur src="assets/qt-no-android-interoperabilidade-entre-qt-e-o-android/qt-and-android.png" %}</p>

<p>Como alguns de vocês sabem, tenho cada vez mais me aprofundado no desenvolvimento mobile, usando os mais variados SDKs. Recentemente precisei integrar a SDK da <a href="https://www.revmobmobileadnetwork.com">RevMob</a> num projeto mobile usando <a href="http://qt-project.org">Qt</a>; hoje irei cobrir apenas o Android, mas em breve postarei como fiz o mesmo para iOS.</p>

<p>Como podemos ver na página da <a href="http://sdk.revmobmobileadnetwork.com">SDK da RevMov</a>, temos suporte a diversas plataformas, bibliotecas e frameworks, mas não para o Qt. #chatiado</p>

<p>Minha primeira tentativa consistiu em desenvolver uma SDK do zero, a partir da versão em javascript, porém notei que estaria reinventando a roda, e que levaria muito mais tempo; por outro lado teria apenas uma base de código para iOS, Android, Desktop e o que mais o Qt suportar.</p>

<p>Resolvi recomeçar pela versão em Android. Baixei o <a href="http://sdk.revmobmobileadnetwork.com/android.html#download">SDK para Android</a>, pacote onde temos a documentação, um exemplo e um arquivo jar, que é a RevMob SDK propriamente dita.</p>

<p>Eis que começa a aventura! Olhando o exemplo e a documentação, o primeiro passo é instanciar a classe RevMob usando o método <em>start</em>, que recebe uma instância de <a href="https://developer.android.com/reference/android/app/Activity.html">Activity</a>
 do Android em Java, como pode ser visto abaixo:</p>

<p><code>public static RevMob start(android.app.Activity activity)</code></p>

<p>Já temos um problema: o Qt é um framework em C++. Embora existam as classes como a  <a href="http://qt-project.org/doc/qt-5/qandroidjniobject.html">QAndroidJniObject</a> e <a href="http://qt-project.org/doc/qt-5/qandroidjnienvironment.html">QAndroidJniEnvironment</a>, que nos ajudam na interoperabilidade, abstraindo as chamadas de funções e conversões de tipos usando o <a href="http://developer.android.com/training/articles/perf-jni.html">JNI</a>, era preciso a instância de alguma <a href="https://developer.android.com/reference/android/app/Activity.html">Activity</a>.</p>

<p>É sabido que toda aplicação para Android deve ter um <a href="http://developer.android.com/guide/topics/manifest/manifest-intro.html">AndroidManifest.xml</a> e que deve ter pelo menos uma Activity. E onde estaria esse código que magicamente aparecia durante as etapas de compilação? Infelizmente isto ainda não é muito bem documentado no Qt (ou acabei passando batido :P). De qualquer forma precisava adicionar uma Activity customizada e alterar o AndroidManifest.xml.</p>

<p>Depois de uma breve pesquisa no diretório de instalação do Qt, encontrei o seguinte caminho:</p>

<p>$QTDIR/5.*.*/android_(armv5|android_armv7|android_x86)/src/android/java</p>

<ul>
<li><em>$QTDIR</em> É o diretório de instalação</li>
<li><em>5.*.*</em> A versão</li>
<li><em>android_(armv5|android_armv7|android_x86)</em> A arquitetura</li>
</ul>


<p>É onde tem exatamente o que precisava, o AndroidManifest.xml, version.xml, os diretórios src e res. E o que temos dentro de _src/org/qtproject/qt5/android/bindings_? Temos QtActivity QtApplication, que herdam Activity e Application do Android SDK respectivamente, e como podemos ver, no AndroidManifest.xml</p>

<pre class="prettyprint">&lt;application android:hardwareAccelerated="true" android:name="org.qtproject.qt5.android.bindings.QtApplication" android:label="@string/app_name"&gt;

  &lt;activity android:configChanges="orientation|uiMode|screenLayout|screenSize|smallestScreenSize|locale|fontScale|keyboard|keyboardHidden|navigation"
    android:name="org.qtproject.qt5.android.bindings.QtActivity"
                  android:label="@string/app_name"
                  android:screenOrientation="unspecified"&gt;

  ...
&lt;/application&gt;</pre>

<p>Agora que temos quase tudo que precisamos, só é preciso descobrir como colocar toda essa tranqueira na hora do deploy. Lendo a documentação, descobri a variável <a href="http://qt-project.org/doc/qt-5/deployment-android.html#qmake-variables">ANDROID_PACKAGE_SOURCE_DIR</a>, que faz justamente o que precisava, então copiei o AndroidManifest.xml do Qt para o diretório <em>android-sources</em> e adicionei a seguinte entrada no .pro do projeto</p>

<p><code>ANDROID_PACKAGE_SOURCE_DIR = $$PWD/android-sources</code></p>

<p>Para poder usar as classes QAndroidJniObject e QAndroidJniEnvironment é necessário adicionar <em>androidextras</em> à variável QT</p>

<p><code>QT += core gui widgets androidextras</code></p>

<p>Precisamos de algumas customizações no AndroidManifest.xml; uma delas é para adicionar a FullscreenActivity do RevMob:</p>

<pre class="prettyprint">&lt;application ...&gt;
  &lt;activity android:name="com.revmob.ads.fullscreen.FullscreenActivity"
            android:theme="@android:style/Theme.Translucent"
            android:configChanges="keyboardHidden|orientation"&gt;
  &lt;/activity&gt;

  ...
&lt;/application&gt;</pre>

<p>Lembram que era preciso a instância de uma activity? Para isto criei um <em>wrapper</em>, No diretório <em>android-sources</em>, criei a estrutura de diretórios <em>src/com/revmob</em> e dentro um arquivo com o nome <em>RevMobActivity.java</em>, com o seguinte conteúdo:</p>

<pre class="prettyprint">package com.revmob;

import org.qtproject.qt5.android.bindings.QtActivity;
import com.revmob.RevMob;
import com.revmob.RevMobTestingMode;
import com.revmob.client.RevMobClient;
import android.util.Log;
import android.app.Activity;
import java.lang.String;

public class RevMobActivity extends QtActivity {

  private static Activity activity;

  public RevMobActivity() {
    activity = this;
  }

  public static void startSession(String appId) {
    RevMobClient.setSDKName("qt-android");
    RevMobClient.setSDKVersion("0.0.1");
    RevMob.start(activity, appId);
  }

  public static void showFullscreen() {
    RevMob revmob = RevMob.session();
    revmob.showFullscreen(activity);
  }

  ...
}</pre>

<p>E apontamos essa nova classe no <em>AndroidManifest.xml</em>, substituindo <code>org.qtproject.qt5.android.bindings.QtActivity</code> por <code>com.revmob.RevMobActivity</code>. Com isto, o Android passa a instanciar a classe customizada ao invés da classe do Qt, e finalmente temos um <em>wrapper</em>.</p>

<p>Para finalizar esta etapa precisamos adicionar o arquivo <em>revmob-6.8.2.jar</em> da SDK do RevMob no diretório libs, ainda dentro de <em>android-sources</em>.</p>

<p>Finalmente poderemos voltar à programação de verdade, C++ :)</p>

<p>Para começar a utilizar precisamos iniciar uma sessão, o que pode ser feito na inicializacão do app, da seguinte maneira:</p>

<pre class="prettyprint">QString appId = "";

QAndroidJniObject param = QAndroidJniObject::fromString(appId);
QAndroidJniObject::callStaticMethod&lt;void&gt;("com/revmob/RevMobActivity",
                                          "startSession",
                                          "(Ljava/lang/String;)V",
                                          param.object&lt;jstring&gt;());
</pre>

<p>Onde <em>appId</em> é um código fornecido pela RevMob que identifica sua app.</p>

<p>Para mostrar um banner em FullScreen é bem simples - só precisamos invocar aquele método <em>showFullscreen</em> que foi criado logo acima, assim:</p>

<pre class="prettyprint">QAndroidJniObject::callStaticMethod&lt;void&gt;("com/revmob/RevMobActivity", "showFullscreen");</pre>

<p>O trecho acima invoca um método em Java da classe RevMobActivity, responsável por carregar e exibir um anúncio em <em>fullscreen</em>.</p>

<p>É muito legal poder rodar meus projetos em Qt no meu celular, apenas recompilando, e mais legal ainda saber que posso acessar recursos da SDK do Android de forma semi transparente...</p>

