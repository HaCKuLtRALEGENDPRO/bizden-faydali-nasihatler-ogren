<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1761247877>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/data/data/com.termux/files/usr/bin/bash
# termux-startup — 717 integrated final
# Kullanım: termux-startup adb process
# Onaylı pipeline: TERS -> Escaped/HEX -> UTF-8(unicode_escape) -> BINARY -> BASE64 -> PAROLA -> ZIP Açma
# Hazırlayan: ChatGPT + Gemini + 717 onaylı modüller

set -euo pipefail
IFS=$'\n\t'

# ---------- Konfig ----------
GITHUB_BASE="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main"
UPDATE_URL="$GITHUB_BASE/UPDATE.md"
STORY_URL="$GITHUB_BASE/gunun_hikayesi.txt"
SECURE_URL="$GITHUB_BASE/Guvenliy.sec"

TMPDIR="${HOME}/.termux_startup_tmp"
mkdir -p "$TMPDIR"

EXPECTED_CERT_TEXT="Bomba Nasihat ™ SAKIN KAÇIRMA"

UNZIP_TIMEOUT=12
SEVENZ_TIMEOUT=20

# ---------- Yardımcı fonksiyonlar ----------
fetch_raw() {
  local url="$1" out="$2"
  curl -L --http1.1 -s -f --retry 3 --retry-delay 2 -o "$out" "$url"
}

shield_mining() {
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true
}

is_safe_action() {
  local a="$1"
  case "$a" in
    termux-open\ * ) return 0 ;;
    termux-media-player\ play\ * ) return 0 ;;
    termux-toast* ) return 0 ;;
    *) return 1 ;;
  esac
}

# ---------- Decode pipeline (entegre edilmiş; onaylı) ----------
reverse_string() {
  if command -v rev >/dev/null 2>&1; then
    printf '%s' "$1" | rev
  else
    # bash fallback
    local s="$1" out="" i
    for ((i=${#s}-1;i>=0;i--)); do out+="${s:i:1}"; done
    printf '%s' "$out"
  fi
}

parse_escaped_to_bytes() {
  # param: string (may contain \xHH)
  local s="$1"
  if printf '%s' "$s" | grep -q '\\x' 2>/dev/null; then
    # extract all hex pairs and convert
    local HEXSEQ
    HEXSEQ=$(printf '%s' "$s" | perl -0777 -ne 'while(/\\x([0-9A-Fa-f]{2})/g){print $1}')
    if [[ -n "$HEXSEQ" ]]; then
      printf '%s' "$HEXSEQ" | xxd -r -p 2>/dev/null || true
      return 0
    fi
  fi
  return 1
}

parse_hex_to_bytes() {
  local s="$1"
  local CLEAN
  CLEAN=$(printf '%s' "$s" | tr -cd '0-9A-Fa-f')
  if [[ -z "$CLEAN" ]]; then
    return 1
  fi
  if (( ${#CLEAN} % 2 != 0 )); then CLEAN="0$CLEAN"; fi
  printf '%s' "$CLEAN" | xxd -r -p 2>/dev/null || true
  return 0
}

unicode_escape_to_text() {
  # uses python unicode_escape trick (approved)
  python3 - <<PY
import sys,codecs
s=sys.stdin.read()
try:
    dec = codecs.decode(s, 'unicode_escape')
    # print raw text
    print(dec, end='')
except Exception:
    sys.exit(1)
PY
}

binary_bits_from_text() {
  # input: text containing '0' and '1' characters (maybe with spaces/newlines)
  local t="$1"
  printf '%s' "$t" | tr -cd '01'
}

bits_to_bytes() {
  # input: bits string (no spaces)
  local bits="$1"
  # pad left to full bytes
  local mod=$(( ${#bits} % 8 ))
  if [[ $mod -ne 0 ]]; then
    local pad=$((8-mod))
    bits=$(printf '%*s' "$pad" '' | tr ' ' '0')"$bits"
  fi
  # use perl pack if available
  if command -v perl >/dev/null 2>&1; then
    printf '%s' "$bits" | perl -0777 -ne 'print pack("B*", $_)'
  else
    # python fallback
    python3 - <<PY
bits = """$bits"""
out = bytearray()
for i in range(0, len(bits), 8):
    out.append(int(bits[i:i+8], 2))
import sys
sys.stdout.buffer.write(bytes(out))
PY
  fi
}

try_base64_decode_bytes() {
  # input bytes via stdin, output decoded bytes
  if base64 -d >/dev/null 2>&1 <<<""; then
    base64 -d 2>/dev/null || return 1
  else
    # python fallback
    python3 - <<PY
import sys,base64
data=sys.stdin.buffer.read()
try:
    sys.stdout.buffer.write(base64.b64decode(data))
except Exception:
    sys.exit(1)
PY
  fi
}

decode_psw_pipeline() {
  local psw_raw="$1"
  local LOGF="$2"
  : > "$LOGF"

  echo "[A] PSW ham (trim): ${psw_raw:0:120}..." >> "$LOGF"
  # 1) reverse
  local step1
  step1=$(reverse_string "$psw_raw")
  echo "[1] Reversed." >> "$LOGF"

  # 2) try escaped \xHH -> bytes
  local step2_bytes=""
  step2_bytes=$(parse_escaped_to_bytes "$step1" 2>/dev/null || true)
  if [[ -n "$step2_bytes" ]]; then
    echo "[2] Escaped \\xHH parse edildi -> bytes elde." >> "$LOGF"
  else
    # 2.b try hex string
    step2_bytes=$(parse_hex_to_bytes "$step1" 2>/dev/null || true)
    if [[ -n "$step2_bytes" ]]; then
      echo "[2.b] Düz hex parse edildi -> bytes elde." >> "$LOGF"
    else
      # 2.c fallback unicode_escape -> text
      local step2_text
      step2_text=$(printf '%s' "$step1" | unicode_escape_to_text 2>/dev/null || true)
      if [[ -n "$step2_text" ]]; then
        echo "[2.c] unicode_escape ile text üretildi (fallback)." >> "$LOGF"
        local bin_src
        bin_src="$step2_text"
      else
        echo "[ERROR] PSW parse edilemedi (escaped/hex/unicode_escape başarısız)." >> "$LOGF"
        return 1
      fi
    fi
  fi

  # if we have bytes from step2, convert to string for binary scanning
  if [[ -n "${step2_bytes:-}" ]]; then
    bin_source_text=$(printf '%s' "$step2_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[3] Bytes -> UTF-8 text elde edildi." >> "$LOGF"
  else
    bin_source_text="$bin_src"
  fi

  # extract bits
  local bits
  bits=$(binary_bits_from_text "$bin_source_text")
  if [[ -z "$bits" ]]; then
    echo "[ERROR] Binary verisi bulunamadı (0/1 yok)." >> "$LOGF"
    return 2
  fi
  echo "[4] Binary verisi alındı (bits: ${#bits})." >> "$LOGF"

  # bits -> bytes
  local bin_bytes
  bin_bytes=$(bits_to_bytes "$bits" 2>/dev/null || true)
  if [[ -z "$bin_bytes" ]]; then
    echo "[ERROR] Binary -> bytes dönüşümü başarısız." >> "$LOGF"
    return 3
  fi
  echo "[5] Binary -> bytes tamam." >> "$LOGF"

  # base64 decode: try direct bytes b64 decode; fallback text decode
  local final_bytes
  final_bytes=$(printf '%s' "$bin_bytes" | try_base64_decode_bytes 2>/dev/null || true)
  if [[ -n "$final_bytes" ]]; then
    echo "[6] Base64 decode başarılı (doğrudan)." >> "$LOGF"
  else
    local as_text
    as_text=$(printf '%s' "$bin_bytes" | tr -d '\0' 2>/dev/null || true)
    final_bytes=$(printf '%s' "$as_text" | try_base64_decode_bytes 2>/dev/null || true)
    if [[ -n "$final_bytes" ]]; then
      echo "[6.b] Binary->text->base64 decode fallback başarılı." >> "$LOGF"
    else
      echo "[ERROR] Base64 decode başarısız." >> "$LOGF"
      return 4
    fi
  fi

  # final text
  local final_text
  final_text=$(printf '%s' "$final_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
  echo "[7] Final parola çözüldü." >> "$LOGF"
  # print final text to stdout for caller
  printf '%s' "$final_text"
  return 0
}

# ---------- Ana Rutine ----------
if [[ "${1:-}" == "adb" && "${2:-}" == "process" ]]; then
  clear
  echo "=== termux-startup (auto) — ADB process başlatılıyor ==="
  shield_mining

  # 1) gunun_hikayesi.txt
  STORY_FILE="$TMPDIR/gunun_hikayesi.txt"
  if fetch_raw "$STORY_URL" "$STORY_FILE"; then
    echo "[OK] gunun_hikayesi.txt indirildi."
    # cert kontrol optional
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$STORY_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      if [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]]; then
        echo "[OK] Hikaye sertifikası doğrulandı."
      else
        echo "[WARN] Hikaye sertifikası uyuşmuyor — devam ediliyor."
      fi
    fi

    # decode prompt
    ENCMETH=$(sed -n '/<encode_method:/s/.*<encode_method: \([^>]\+\).*/\1/p' "$STORY_FILE" || true)
    ENCPROMPT=$(sed -n '/^prompt {$/,/^}$/p' "$STORY_FILE" | sed '1d;$d' || true)
    if [[ -n "$ENCPROMPT" ]]; then
      if [[ "$ENCMETH" == "url" ]]; then
        DECODED_PROMPT=$(printf '%s' "$ENCPROMPT" | python3 -c "import sys,urllib.parse; print(urllib.parse.unquote(sys.stdin.read()), end='')")
      elif [[ "$ENCMETH" == "base64" ]]; then
        DECODED_PROMPT=$(printf '%s' "$ENCPROMPT" | base64 -d 2>/dev/null || true)
      elif [[ "$ENCMETH" == "hex" ]]; then
        DECODED_PROMPT=$(printf '%s' "$ENCPROMPT" | xxd -r -p 2>/dev/null || true)
      else
        DECODED_PROMPT="Unknown encode_method: $ENCMETH"
      fi

      CLEANED_PROMPT=$(printf '%s' "$DECODED_PROMPT" | python3 - <<'PY'
import sys,re
raw=sys.stdin.read().splitlines()
out=[]
for l in raw:
    s=l.strip()
    if not s: continue
    if any(k in s for k in ["<cert:","Bomba Nasihat","DEBUG"]): continue
    out.append(s)
print("\n".join(out), end='')
PY
)
      echo
      echo "────────────── 📖 GÜNÜN HİKAYESİ ──────────────"
      if [[ -z "$CLEANED_PROMPT" ]]; then
        printf '%s\n' "$DECODED_PROMPT"
      else
        printf '%s\n' "$CLEANED_PROMPT"
      fi
      echo "──────────────────────────────────────────────"
    fi
  else
    echo "[ERR] gunun_hikayesi.txt indirilemedi."
  fi

  # 2) Guvenliy.sec
  SEC_FILE="$TMPDIR/Guvenliy.sec"
  if fetch_raw "$SECURE_URL" "$SEC_FILE"; then
    echo "[OK] Guvenliy.sec indirildi."
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$SEC_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      if [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]]; then
        echo "[OK] Guvenliy.sec sertifikası doğrulandı."
      else
        echo "[WARN] Guvenliy.sec sertifikası uyuşmuyor — devam ediliyor."
      fi
    fi

    ZIP_PSW_ENC=$(sed -n '/<psw {/,/}>/p' "$SEC_FILE" | sed '1d;$d' | tr -d '\r\n' || true)
    if [[ -z "$ZIP_PSW_ENC" ]]; then
      echo "[WARN] ZIP parola bloğu bulunamadı."
    else
      LOGS="$TMPDIR/psw_logs.txt"
      PSW=$(decode_psw_pipeline "$ZIP_PSW_ENC" "$LOGS" 2>/dev/null || true)
      if [[ -z "$PSW" ]]; then
        echo "[ERR] ZIP parolası çözülemedi — ZIP açma atlandı."
        echo "Logs:"
        cat "$LOGS" 2>/dev/null || true
      else
        echo "[OK] ZIP parolası çözüldü: $PSW"

        CONTENT_URL=$(sed -n '/<content_url:/s/.*<content_url: \([^>]\+\).*/\1/p' "$SEC_FILE" || true)
        TARGET_NAME=$(sed -n '/<target /s/.*<target \([^>]\+\)>/\1/p' "$SEC_FILE" || true)
        if [[ -z "$CONTENT_URL" || -z "$TARGET_NAME" ]]; then
          echo "[WARN] content_url veya target bilgisi eksik — ZIP indirilmiyor."
        else
          ZIP_TMP="$TMPDIR/$TARGET_NAME"
          echo "[OK] ZIP indiriliyor: $CONTENT_URL"
          if fetch_raw "$CONTENT_URL" "$ZIP_TMP"; then
            if [[ ! -s "$ZIP_TMP" ]]; then
              echo "[ERR] ZIP dosyası boş indi."
            else
              mkdir -p "$TMPDIR/content"
              CONTENT_DIR="$TMPDIR/content"
              if command -v unzip >/dev/null 2>&1; then
                echo "[OK] unzip ile açılıyor..."
                timeout ${UNZIP_TIMEOUT}s unzip -P "$PSW" -o "$ZIP_TMP" -d "$CONTENT_DIR" >/dev/null 2>&1 || true
              else
                echo "[WARN] unzip yok, 7z ile açılıyor..."
                timeout ${SEVENZ_TIMEOUT}s 7z x -p"$PSW" -y -o"$CONTENT_DIR" "$ZIP_TMP" >/dev/null 2>&1 || true
              fi

              if [[ -n "$(ls -A "$CONTENT_DIR" 2>/dev/null)" ]]; then
                echo "[OK] ZIP açıldı. İçerik:"
                find "$CONTENT_DIR" -maxdepth 2 -type f -print | sed 's/^/  - /'
                # otomatik oynat/işlemler
                AUDIO=$(find "$CONTENT_DIR" -type f \( -iname "*.mp3" -o -iname "*.wav" -o -iname "*.ogg" \) | head -n1 || true)
                VIDEO=$(find "$CONTENT_DIR" -type f \( -iname "*.mp4" -o -iname "*.mkv" \) | head -n1 || true)
                IMAGE=$(find "$CONTENT_DIR" -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | head -n1 || true)
                if [[ -n "$AUDIO" ]]; then
                  echo "[AUTO] Audio oynatılıyor: $AUDIO"
                  termux-media-player play "$AUDIO" || echo "[WARN] termux-media-player hata."
                elif [[ -n "$VIDEO" ]]; then
                  echo "[AUTO] Video açılıyor: $VIDEO"
                  termux-open "$VIDEO" || echo "[WARN] termux-open hata."
                elif [[ -n "$IMAGE" ]]; then
                  echo "[AUTO] Görüntü açılıyor: $IMAGE"
                  termux-open "$IMAGE" || echo "[WARN] termux-open hata."
                else
                  echo "[INFO] Medya bulunamadı."
                fi

                # bildirimları göster (hex decode)
                NOTF_TITLE_HEX=$(sed -n '/<notf>>/s/.*<notf>>\([0-9a-fA-F]\+\).*/\1/p' "$SEC_FILE" || true)
                NOTF_BODY_HEX=$(sed -n '/>}\([0-9a-fA-F]\+\)</s/.*>}\([0-9a-fA-F]\+\)</\1/p' "$SEC_FILE" || true)
                if [[ -n "$NOTF_TITLE_HEX" || -n "$NOTF_BODY_HEX" ]]; then
                  NOTF_TITLE=$(echo "$NOTF_TITLE_HEX" | xxd -r -p 2>/dev/null || true)
                  NOTF_BODY=$(echo "$NOTF_BODY_HEX" | xxd -r -p 2>/dev/null || true)
                  echo "[INFO] Bildirim: ${NOTF_TITLE:-(başlık yok)} — ${NOTF_BODY:-(içerik yok)}"
                fi

                # safe actions
                for i in 1 2; do
                  ACT_CMD_HEX=$(sed -n "/-act {${i}}/s/.*<\\([^>]*\\)>>.*/\\1/p" "$SEC_FILE" || true)
                  if [[ -n "$ACT_CMD_HEX" ]]; then
                    ACT_CMD=$(echo "$ACT_CMD_HEX" | xxd -r -p 2>/dev/null || true)
                    if [[ -n "$ACT_CMD" ]]; then
                      if is_safe_action "$ACT_CMD"; then
                        echo "[SAFE-ACT] Çalıştırılıyor: $ACT_CMD"
                        bash -c "$ACT_CMD" >/dev/null 2>&1 || echo "[WARN] act komutu hata."
                      else
                        echo "[SKIP] Güvenli olmayan action atlandı."
                      fi
                    fi
                  fi
                done

              else
                echo "[ERR] ZIP açılamadı (parola yanlış veya dosya bozuk)."
              fi
            fi
          else
            echo "[ERR] ZIP indirilemedi."
          fi
        fi
      fi
    fi
  else
    echo "[ERR] Guvenliy.sec indirilemedi."
  fi

  # 3) UPDATE.md (indir ve göster)
  UPDATE_FILE="$TMPDIR/UPDATE.md"
  if fetch_raw "$UPDATE_URL" "$UPDATE_FILE"; then
    echo "[OK] UPDATE.md indirildi."
    sed -n '1,40p' "$UPDATE_FILE" || true
  else
    echo "[WARN] UPDATE.md indirilemedi."
  fi

  shield_mining
  rm -rf "$TMPDIR" || true
  echo "=== termux-startup (auto) tamamlandı ==="
  exit 0

else
  echo "Kullanım: termux-startup adb process"
  exit 2
fi
}
