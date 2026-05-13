require "import"
import "com.androlua.Http"
import "cjson"
import "com.androlua.LuaDialog"
import "android.widget.*"
import "android.view.*"
import "android.text.InputFilter"
import "android.content.Context"
import "android.media.MediaPlayer"
import "android.util.Base64"
import "android.os.*"
import "android.graphics.Typeface"
import "java.io.*"

local context = activity or service
local mainHandler = Handler(Looper.getMainLooper())
local player = MediaPlayer()
local lastGeneratedPath = ""

local sp = context.getSharedPreferences("gemini_tts_prefs", Context.MODE_PRIVATE)
local googleApiKey = sp.getString("api_key", "")
local isMultiMode = sp.getBoolean("multi_mode", false)

local rootPath = Environment.getExternalStorageDirectory().toString().."/Google Gemini TTS/"
local folder = File(rootPath)
if not folder.exists() then folder.mkdirs() end

local VOICE_LIST = {"Puck", "Kore", "Charon", "Zephyr", "Fenrir", "Leda", "Orus", "Aoede", "Callirrhoe", "Autonoe", "Enceladus", "Iapetus", "Umbriel", "Algieba", "Despina", "Erinome", "Algenib", "Rasalgethi", "Laomedeia", "Achernar", "Alnilam", "Schedar", "Gacrux", "Pulcherrima"}
local EMOTION_LIST = {"Neutral", "Happy", "Sad", "Angry", "Excited", "Friendly", "Custom..."}

local function setSafeMargins(view, top, bottom)
    local lp = view.getLayoutParams()
    if lp and luajava.instanceof(lp, ViewGroup.MarginLayoutParams) then
        lp.setMargins(0, top or 0, 0, bottom or 0)
        view.setLayoutParams(lp)
    end
end

-- جدید ترین Gemini 2.0 Flash ماڈل کا استعمال
function callGeminiTTS(dataArray, callback)
    -- ماڈل کو gemini-2.0-flash پر اپڈیٹ کر دیا گیا ہے
    local url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=" .. googleApiKey
    
    local prompt = "Task: Act as a high-quality TTS engine. Convert the following text into a Base64 encoded audio string. Output ONLY the raw Base64 string without any markdown, labels, or extra text.\n\n"
    for _, item in ipairs(dataArray) do
        prompt = prompt .. string.format("[Voice: %s, Emotion: %s] Text: %s\n", item.voice, item.emotion, item.text)
    end

    local postBody = {
        contents = {{ parts = {{ text = prompt }} }},
        generationConfig = {
            temperature = 0.1,
            maxOutputTokens = 8192,
        }
    }

    local jsonData = cjson.encode(postBody)

    Http.post(url, jsonData, {
        ["Content-Type"] = "application/json"
    }, function(code, res)
        if code == 200 then
            local success, decoded = pcall(cjson.decode, res)
            if success and decoded.candidates and decoded.candidates[1].content.parts[1].text then
                local rawOutput = decoded.candidates[1].content.parts[1].text
                -- کلین اپ: تمام فالتو سپیسز اور فارمیٹنگ ختم کرنا
                local audioBase64 = rawOutput:gsub("`", ""):gsub("base64", ""):gsub("%s+", ""):match("([A-Za-z0-9+/=]+)")
                
                if audioBase64 and #audioBase64 > 50 then
                    local audioBytes = Base64.decode(audioBase64, Base64.DEFAULT)
                    local fos = FileOutputStream(lastGeneratedPath)
                    fos.write(audioBytes)
                    fos.close()
                    callback(true)
                else
                    callback(false, "Invalid Audio Data. Gemini returned: " .. tostring(rawOutput))
                end
            else
                callback(false, "Response structure mismatch.")
            end
        else
            callback(false, "API Error " .. code .. ": " .. tostring(res))
        end
    end)
end

function showApiSettings()
    local views = {}
    local layout = { LinearLayout, orientation = "vertical", padding = "24dp", { EditText, id = "apiInput", hint = "Enter Gemini API key", layout_width = "fill", backgroundColor = "#F5F5F5", padding = "12dp" }, { Button, id = "saveBtn", text = "SAVE & CLOSE", layout_width = "fill", backgroundColor = "#4CAF50", textColor = "#FFFFFF", layout_marginTop = "15dp" } }
    local dlg = LuaDialog(context).setView(loadlayout(layout, views))
    views.apiInput.setText(googleApiKey)
    views.saveBtn.onClick = function()
        googleApiKey = views.apiInput.getText().toString()
        sp.edit().putString("api_key", googleApiKey).apply()
        dlg.dismiss()
    end
    dlg.show()
end

function showMain()
    local views = {}
    local dialogueCount = 1
    local dialogueInputs = {}

    local scrollLayout = { ScrollView, layout_width = "fill", { LinearLayout, orientation = "vertical", padding = "25dp", layout_width = "fill", { TextView, id = "mainHeader", text = isMultiMode and "Dialogue Mode Active" or "Single Voice Mode", textSize = 16, textColor = "#2E7D32", typeface = Typeface.DEFAULT_BOLD }, { CheckBox, id = "multiVoiceToggle", text = "Enable Dialogue (Multi-Voice)", checked = isMultiMode }, { LinearLayout, id = "dynamicContainer", orientation = "vertical", layout_width = "fill" }, { Button, id = "addMoreBtn", text = "ADD DIALOGUE LINE (+)", layout_width = "fill", backgroundColor = "#009688", textColor = "#FFFFFF", visibility = isMultiMode and View.VISIBLE or View.GONE }, { Button, id = "generateBtn", text = "GENERATE AUDIO", layout_width = "fill", backgroundColor = "#2196F3", textColor = "#FFFFFF" }, { LinearLayout, id = "resultLayout", orientation = "horizontal", layout_width = "fill", visibility = View.GONE, { Button, id = "playBtn", text = "PLAY", layout_width = "0dp", layout_weight = "1", backgroundColor = "#4CAF50", textColor = "#FFFFFF" }, { Button, id = "saveFileBtn", text = "SAVE PERMANENTLY", layout_width = "0dp", layout_weight = "1", backgroundColor = "#FF9800", textColor = "#FFFFFF" } }, { Button, id = "apiBtn", text = "API SETTING", layout_width = "fill", backgroundColor = "#9C27B0", textColor = "#FFFFFF" }, { Button, id = "exitBtn", text = "EXIT", layout_width = "fill", backgroundColor = "#D32F2F", textColor = "#FFFFFF" } } }
    local dlg = LuaDialog(context).setView(loadlayout(scrollLayout, views)).setCancelable(false)

    setSafeMargins(views.mainHeader, 0, 10)
    setSafeMargins(views.multiVoiceToggle, 0, 20)
    setSafeMargins(views.generateBtn, 20, 10)

    local function createDialogueBlock(index)
        local idSuffix = tostring(index)
        local dViews = {}
        local block = { LinearLayout, orientation = "vertical", layout_width = "fill", padding = "10dp", backgroundColor = "#F9F9F9", { TextView, text = "Line #" .. idSuffix, textColor = "#1976D2", typeface = Typeface.DEFAULT_BOLD, visibility = (index > 1 or views.multiVoiceToggle.isChecked()) and View.VISIBLE or View.GONE }, { Spinner, id = "voiceSpin" .. idSuffix, layout_width = "fill" }, { Spinner, id = "emotionSpin" .. idSuffix, layout_width = "fill" }, { EditText, id = "textInput" .. idSuffix, hint = "Type message here...", layout_width = "fill", layout_height = "80dp", gravity = Gravity.TOP, backgroundColor = "#FFFFFF", padding = "8dp" }, { TextView, id = "charCounter" .. idSuffix, text = "0 / 2000", textColor = "#757575", gravity = "right" } }
        views.dynamicContainer.addView(loadlayout(block, dViews))
        dViews["voiceSpin"..idSuffix].setAdapter(ArrayAdapter(context, android.R.layout.simple_spinner_item, VOICE_LIST))
        dViews["emotionSpin"..idSuffix].setAdapter(ArrayAdapter(context, android.R.layout.simple_spinner_item, EMOTION_LIST))
        dViews["textInput"..idSuffix].addTextChangedListener({ onTextChanged = function(s) dViews["charCounter"..idSuffix].setText(s.length() .. " / 2000") end })
        dialogueInputs[index] = dViews
    end

    createDialogueBlock(1)

    views.multiVoiceToggle.setOnCheckedChangeListener(CompoundButton.OnCheckedChangeListener{ onCheckedChanged = function(b, isChecked)
        sp.edit().putBoolean("multi_mode", isChecked).apply()
        views.mainHeader.setText(isChecked and "Dialogue Mode Active" or "Single Voice Mode")
        views.addMoreBtn.setVisibility(isChecked and View.VISIBLE or View.GONE)
        views.dynamicContainer.removeAllViews()
        dialogueInputs, dialogueCount = {}, 1
        createDialogueBlock(1)
    end })

    views.addMoreBtn.onClick = function() dialogueCount = dialogueCount + 1 createDialogueBlock(dialogueCount) end

    views.generateBtn.onClick = function()
        if googleApiKey == "" then print("API Key Missing") return end
        views.generateBtn.setText("GENERATING...")
        lastGeneratedPath = rootPath .. "last_speech.mp3"
        local data = {}
        for i=1, dialogueCount do
            local v = dialogueInputs[i]
            local t = v["textInput"..i].getText().toString()
            if t ~= "" then
                table.insert(data, { voice = VOICE_LIST[v["voiceSpin"..i].getSelectedItemPosition() + 1], emotion = EMOTION_LIST[v["emotionSpin"..i].getSelectedItemPosition() + 1], text = t })
            end
        end
        if #data == 0 then print("No text provided") views.generateBtn.setText("GENERATE AUDIO") return end
        
        callGeminiTTS(data, function(ok, err) mainHandler.post(Runnable{run=function()
            if ok then views.generateBtn.setText("SUCCESS!") views.resultLayout.setVisibility(View.VISIBLE) else views.generateBtn.setText("FAILED (Try Again)") print(err) end
        end}) end)
    end

    views.playBtn.onClick = function()
        if player.isPlaying() then player.pause() views.playBtn.setText("RESUME") else
            if views.playBtn.getText().toString() == "RESUME" then player.start() else
                player.reset() player.setDataSource(lastGeneratedPath) player.prepare() player.start()
            end
            views.playBtn.setText("PAUSE")
        end
    end

    views.saveFileBtn.onClick = function()
        local edit = EditText(context).setHint("File name (e.g. greeting)")
        LuaDialog(context).setTitle("Save MP3").setView(edit).setPositiveButton("SAVE", {onClick=function()
            local n = edit.getText().toString()
            if n ~= "" then File(lastGeneratedPath).renameTo(File(rootPath .. n .. ".mp3")) print("File saved to folder!") end
        end}).show()
    end

    views.apiBtn.onClick = function() showApiSettings() end
    views.exitBtn.onClick = function() dlg.dismiss() end
    dlg.show()
end

showMain()