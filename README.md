name: Build Jarvis APK
on:
  workflow_dispatch:
  push:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create project files
        run: |
          mkdir -p app/src/main/java/com/jarvis/app
          cat > settings.gradle.kts <<'EOF'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "Jarvis"
          include(":app")
          EOF
          cat > build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF
          cat > gradle.properties <<'EOF'
          org.gradle.jvmargs=-Xmx2g
          EOF
          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.jarvis.app"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.jarvis.app"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.RECORD_AUDIO" />
              <uses-permission android:name="com.android.alarm.permission.SET_ALARM" />
              <queries>
                  <intent><action android:name="android.speech.RecognitionService" /></intent>
                  <intent><action android:name="android.intent.action.TTS_SERVICE" /></intent>
              </queries>
              <application
                  android:label="Jarvis"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:windowSoftInputMode="adjustResize">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          EOF
          cat > app/src/main/java/com/jarvis/app/MainActivity.kt <<'EOF'
          package com.jarvis.app

          import android.Manifest
          import android.app.Activity
          import android.app.AlertDialog
          import android.content.Context
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.graphics.Color
          import android.net.Uri
          import android.os.Bundle
          import android.provider.AlarmClock
          import android.speech.RecognitionListener
          import android.speech.RecognizerIntent
          import android.speech.SpeechRecognizer
          import android.speech.tts.TextToSpeech
          import android.view.ViewGroup
          import android.widget.Button
          import android.widget.EditText
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.TextView
          import org.json.JSONArray
          import org.json.JSONObject
          import java.net.HttpURLConnection
          import java.net.URL
          import java.text.DateFormat
          import java.util.Date
          import java.util.Locale

          class MainActivity : Activity(), TextToSpeech.OnInitListener {
              private lateinit var tts: TextToSpeech
              private var recognizer: SpeechRecognizer? = null
              private lateinit var logView: TextView
              private lateinit var scroll: ScrollView
              private lateinit var statusView: TextView
              private lateinit var input: EditText
              private lateinit var langBtn: Button
              private var ml = false
              private var busy = false
              private val history = JSONArray()
              private val prefs by lazy { getSharedPreferences("jarvis", Context.MODE_PRIVATE) }
              private val accent = Color.parseColor("#27D3FF")

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              private fun btn(label: String, onClick: () -> Unit) = Button(this).apply {
                  text = label
                  setOnClickListener { onClick() }
              }

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  tts = TextToSpeech(this, this)
                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setBackgroundColor(Color.parseColor("#050B14"))
                      setPadding(dp(14), dp(12), dp(14), dp(14))
                  }
                  val header = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
                  header.addView(TextView(this).apply {
                      text = "J.A.R.V.I.S"; textSize = 18f; setTextColor(accent)
                      layoutParams = LinearLayout.LayoutParams(0, ViewGroup.LayoutParams.WRAP_CONTENT, 1f)
                  })
                  langBtn = btn("English") {
                      ml = !ml
                      langBtn.text = if (ml) "മലയാളം" else "English"
                  }
                  header.addView(langBtn)
                  header.addView(btn("Key") { askKey() })
                  root.addView(header)

                  logView = TextView(this).apply {
                      textSize = 15f; setTextColor(Color.parseColor("#D9F3FF"))
                      text = "Tap the mic and speak, or type below."
                  }
                  scroll = ScrollView(this).apply {
                      layoutParams = LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f)
                      addView(logView)
                  }
                  root.addView(scroll)

                  statusView = TextView(this).apply { setTextColor(Color.parseColor("#6F93A8")) }
                  root.addView(statusView)
                  root.addView(btn("🎙  Tap to talk") { listen() }.apply { textSize = 20f })

                  val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
                  input = EditText(this).apply {
                      hint = "Ask Jarvis…"; setTextColor(Color.WHITE); setHintTextColor(Color.GRAY)
                      layoutParams = LinearLayout.LayoutParams(0, ViewGroup.LayoutParams.WRAP_CONTENT, 1f)
                  }
                  row.addView(input)
                  row.addView(btn("Send") {
                      val q = input.text.toString()
                      input.setText("")
                      ask(q)
                  })
                  root.addView(row)
                  setContentView(root)

                  if (prefs.getString("key", "").isNullOrEmpty()) askKey()
              }

              override fun onInit(s: Int) {
                  if (s == TextToSpeech.SUCCESS) tts.language = Locale("en", "IN")
              }

              override fun onDestroy() {
                  tts.shutdown()
                  recognizer?.destroy()
                  super.onDestroy()
              }

              private fun askKey() {
                  val e = EditText(this).apply { hint = "sk-ant-..." }
                  AlertDialog.Builder(this)
                      .setTitle("Anthropic API key")
                      .setView(e)
                      .setPositiveButton("Save") { _, _ ->
                          prefs.edit().putString("key", e.text.toString().trim()).apply()
                      }
                      .setNegativeButton("Cancel", null)
                      .show()
              }

              private fun append(who: String, text: String) {
                  logView.append("\n\n$who: $text")
                  scroll.post { scroll.fullScroll(ScrollView.FOCUS_DOWN) }
              }

              private fun speak(text: String) {
                  tts.language = if (ml) Locale("ml", "IN") else Locale("en", "IN")
                  tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, "jarvis")
              }

              private fun listen() {
                  if (checkSelfPermission(Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), 1)
                      return
                  }
                  if (!SpeechRecognizer.isRecognitionAvailable(this)) {
                      statusView.text = "Speech recognition is not available on this phone."
                      return
                  }
                  tts.stop()
                  recognizer?.destroy()
                  recognizer = SpeechRecognizer.createSpeechRecognizer(this).apply {
                      setRecognitionListener(object : RecognitionListener {
                          override fun onReadyForSpeech(p: Bundle?) { statusView.text = "Listening…" }
                          override fun onBeginningOfSpeech() {}
                          override fun onRmsChanged(v: Float) {}
                          override fun onBufferReceived(b: ByteArray?) {}
                          override fun onEndOfSpeech() {}
                          override fun onError(error: Int) { statusView.text = "Didn't catch that. Tap and try again." }
                          override fun onPartialResults(r: Bundle?) {
                              val t = r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
                              if (t != null) statusView.text = t
                          }
                          override fun onEvent(t: Int, p: Bundle?) {}
                          override fun onResults(r: Bundle?) {
                              val t = r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
                              statusView.text = ""
                              if (t != null) ask(t)
                          }
                      })
                  }
                  val i = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
                      .putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                      .putExtra(RecognizerIntent.EXTRA_LANGUAGE, if (ml) "ml-IN" else "en-IN")
                      .putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
                  recognizer?.startListening(i)
              }

              override fun onRequestPermissionsResult(code: Int, perms: Array<out String>, res: IntArray) {
                  super.onRequestPermissionsResult(code, perms, res)
                  if (res.isNotEmpty() && res[0] == PackageManager.PERMISSION_GRANTED) listen()
              }

              private fun ask(q: String) {
                  if (busy || q.isBlank()) return
                  if (prefs.getString("key", "").isNullOrEmpty()) { askKey(); return }
                  busy = true
                  statusView.text = "Thinking…"
                  append("You", q)
                  if (history.length() > 30) { while (history.length() > 0) history.remove(0) }
                  history.put(JSONObject().put("role", "user").put("content", q))
                  Thread {
                      var reply: String
                      try {
                          reply = converse()
                      } catch (e: Exception) {
                          reply = "Error: ${e.message}"
                      }
                      runOnUiThread {
                          append("Jarvis", reply)
                          statusView.text = ""
                          speak(reply)
                          busy = false
                      }
                  }.start()
              }

              private fun converse(): String {
                  for (step in 0 until 6) {
                      val resp = callApi()
                      val content = resp.getJSONArray("content")
                      history.put(JSONObject().put("role", "assistant").put("content", content))
                      if (resp.optString("stop_reason") != "tool_use") {
                          val sb = StringBuilder()
                          for (i in 0 until content.length()) {
                              val b = content.getJSONObject(i)
                              if (b.getString("type") == "text") sb.append(b.getString("text"))
                          }
                          return sb.toString().trim()
                      }
                      val results = JSONArray()
                      for (i in 0 until content.length()) {
                          val b = content.getJSONObject(i)
                          if (b.getString("type") == "tool_use") {
                              val out = runTool(b.getString("name"), b.getJSONObject("input"))
                              results.put(
                                  JSONObject().put("type", "tool_result")
                                      .put("tool_use_id", b.getString("id")).put("content", out)
                              )
                          }
                      }
                      history.put(JSONObject().put("role", "user").put("content", results))
                  }
                  return "Sorry, that took too many steps."
              }

              private fun systemPrompt(): String {
                  val mem = prefs.getString("mem", "") ?: ""
                  return "You are JARVIS, a witty, polite and efficient personal AI assistant like the one in Iron Man. " +
                      "Keep replies short (1-3 sentences) because they are spoken aloud. No markdown, no emojis. " +
                      "Reply in the language the user speaks (English, Malayalam or Manglish). " +
                      "Use tools when helpful, and use remember when the user tells you something personal worth keeping.\n" +
                      "Known facts about the user:" + (if (mem.isBlank()) " none yet." else mem)
              }

              private fun props(vararg p: Pair<String, String>) = JSONObject().apply {
                  p.forEach { put(it.first, JSONObject().put("type", it.second)) }
              }

              private fun tool(name: String, desc: String, props: JSONObject, req: List<String>) =
                  JSONObject().put("name", name).put("description", desc).put(
                      "input_schema",
                      JSONObject().put("type", "object").put("properties", props).put("required", JSONArray(req))
                  )

              private fun tools() = JSONArray()
                  .put(tool("get_time", "Get the current local date and time.", props(), listOf()))
                  .put(tool("set_alarm", "Set an alarm at a clock time (24-hour).", props("hour" to "integer", "minute" to "integer", "label" to "string"), listOf("hour")))
                  .put(tool("set_timer", "Start a countdown timer.", props("seconds" to "integer", "label" to "string"), listOf("seconds")))
                  .put(tool("open_url", "Open a website or search in the browser.", props("url" to "string"), listOf("url")))
                  .put(tool("call_number", "Open the phone dialer with a number.", props("number" to "string"), listOf("number")))
                  .put(tool("remember", "Save a fact about the user for future chats.", props("fact" to "string"), listOf("fact")))

              private fun launch(i: Intent) = runOnUiThread {
                  try { startActivity(i) } catch (e: Exception) { statusView.text = "Can't open that." }
              }

              private fun runTool(name: String, a: JSONObject): String = try {
                  when (name) {
                      "get_time" -> DateFormat.getDateTimeInstance(DateFormat.FULL, DateFormat.SHORT).format(Date())
                      "set_alarm" -> {
                          launch(
                              Intent(AlarmClock.ACTION_SET_ALARM)
                                  .putExtra(AlarmClock.EXTRA_HOUR, a.getInt("hour"))
                                  .putExtra(AlarmClock.EXTRA_MINUTES, a.optInt("minute", 0))
                                  .putExtra(AlarmClock.EXTRA_MESSAGE, a.optString("label", "Jarvis"))
                                  .putExtra(AlarmClock.EXTRA_SKIP_UI, true)
                          )
                          "Alarm set."
                      }
                      "set_timer" -> {
                          launch(
                              Intent(AlarmClock.ACTION_SET_TIMER)
                                  .putExtra(AlarmClock.EXTRA_LENGTH, a.getInt("seconds"))
                                  .putExtra(AlarmClock.EXTRA_MESSAGE, a.optString("label", "Jarvis"))
                                  .putExtra(AlarmClock.EXTRA_SKIP_UI, true)
                          )
                          "Timer started."
                      }
                      "open_url" -> { launch(Intent(Intent.ACTION_VIEW, Uri.parse(a.getString("url")))); "Opened." }
                      "call_number" -> { launch(Intent(Intent.ACTION_DIAL, Uri.parse("tel:" + a.getString("number")))); "Dialer opened." }
                      "remember" -> {
                          val old = prefs.getString("mem", "") ?: ""
                          prefs.edit().putString("mem", old + "\n- " + a.getString("fact")).apply()
                          "Saved."
                      }
                      else -> "Unknown tool."
                  }
              } catch (e: Exception) {
                  "Failed: ${e.message}"
              }

              private fun callApi(): JSONObject {
                  val body = JSONObject()
                      .put("model", "claude-sonnet-5-5")
                      .put("max_tokens", 1024)
                      .put("system", systemPrompt())
                      .put("tools", tools())
                      .put("messages", history)
                  val c = URL("https://api.anthropic.com/v1/messages").openConnection() as HttpURLConnection
                  c.requestMethod = "POST"
                  c.connectTimeout = 20000
                  c.readTimeout = 60000
                  c.doOutput = true
                  c.setRequestProperty("content-type", "application/json")
                  c.setRequestProperty("x-api-key", prefs.getString("key", "") ?: "")
                  c.setRequestProperty("anthropic-version", "2023-06-01")
                  c.outputStream.use { it.write(body.toString().toByteArray()) }
                  val code = c.responseCode
                  val stream = if (code in 200..299) c.inputStream else c.errorStream
                  val text = stream.bufferedReader().use { it.readText() }
                  if (code !in 200..299) throw Exception("API $code: " + text.take(200))
                  return JSONObject(text)
              }
          }
          EOF
      - uses: actions/setup-java@v4
        with
