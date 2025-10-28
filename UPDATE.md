<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1761247877>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/data/data/com.termux/files/usr/bin/bash
# termux-startup — 717 auto-run final
# Kullanım: termux-startup adb process
# NOT: Bu sürüm otomatik çalışır, runtime/system dosyası yazmaz.
# Sorumluluk kullanıcıya aittir (sen onay verdin).

set -euo pipefail
IFS=$'\n\t'

# ------------- Konfig -------------
GITHUB_BASE="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main"
UPDATE_URL="$GITHUB_BASE/UPDATE.md"
STORY_URL="$GITHUB_BASE/gunun_hikayesi.txt"
SECURE_URL="$GITHUB_BASE/Guvenliy.sec"

TMPDIR="${HOME}/.termux_startup_tmp"
mkdir -p "$TMPDIR"

EXPECTED_CERT_TEXT="Bomba Nasihat ™ SAKIN KAÇIRMA"

UNZIP_TIMEOUT=12    # unzip için saniye
SEVENZ_TIMEOUT=20   # 7z için saniye

# Whitelist: -act komutları buradaki desenlere uyuyorsa çalıştırılır
is_safe_action() {
  local a="$1"
  case "$a" in
    termux-open\ * ) return 0 ;;
    termux-media-player\ play\ * ) return 0 ;;
    termux-toast* ) return 0 ;;
    *) return 1 ;;
  esac
}

# Basit, güvenli indirme (HTTP/1.1 ile)
fetch_raw() {
  local url="$1" out="$2"
  curl -L --http1.1 -s -f --retry 3 --retry-delay 2 -o "$out" "$url"
}

# URL decode helper (stdin)
urldecode_py(){ python3 - <<'PY' 
import sys,urllib.parse
print(urllib.parse.unquote(sys.stdin.read()), end='')
PY
}

# Temizleyici (ters düzeltme + çöp atma)
clean_text() {
  python3 - <<'PY'
import sys,re
raw = sys.stdin.read().splitlines()
vowels = set("aeıioöuüAEIİOÖUÜ")
def vowel_ratio(s):
    letters=[c for c in s if c.isalpha()]
    if not letters: return 0.0
    return sum(1 for c in letters if c in vowels)/len(letters)
out=[]
for L in raw:
    s=L.strip()
    if not s:
        out.append("")
        continue
    if any(k in s for k in ["<cert:", "DEBUG", "Debug:", "Bomba Nasihat", "SAKIN KAÇIRMA"]):
        continue
    printable = sum(1 for c in s if 32 <= ord(c) <= 126)
    if printable / max(1,len(s)) < 0.6:
        continue
    rev = s[::-1]
    if vowel_ratio(rev) > vowel_ratio(s) + 0.15 and re.search(r'\w{3,}', rev):
        candidate = rev
    else:
        candidate = s
    if len(re.sub(r'\W+','',candidate)) < 2:
        continue
    out.append(candidate)
sys.stdout.write("\n".join(out))
PY
}

# Mining/şüpheli süreçleri kapat (güvenlik)
shield_mining() {
  pkill -f "minerd|mining|xmrig" >/dev/null 2>&1 || true
}

# PSW decode pipeline (reversed hex/escaped -> bytes -> utf8 -> binary -> base64 -> final)
decode_psw_pipeline() {
  local psw_raw="$1"
  local logs_file="$2"
  : > "$logs_file"
  echo "[A] Orijinal PSW ham: ${psw_raw:0:120}..." >> "$logs_file"

  # 1) reverse
  local step1
  step1=$(printf '%s' "$psw_raw" | rev)
  echo "[1] Reversed." >> "$logs_file"

  # 2) try parse escaped \xHH
  local step2_bytes=""
  if printf '%s' "$step1" | grep -q '\\x'; then
    # extract hex pairs
    hexs=$(printf '%s' "$step1" | sed -n 's/.*\\x\([0-9A-Fa-f][0-9A-Fa-f]\).*/\\x\1/p' || true)
    # safer: use perl to extract all \xHH
    hexs=$(printf '%s' "$step1" | perl -ne 'while(/\\x([0-9A-Fa-f]{2})/g){print "$1"}' || true)
    if [[ -n "$hexs" ]]; then
      # build byte string
      step2_bytes=$(printf '%s' "$hexs" | xxd -r -p 2>/dev/null || true)
      if [[ -n "$step2_bytes" ]]; then
        echo "[2] Escaped \\xHH formatı bulundu -> bytes elde edildi." >> "$logs_file"
      else
        echo "[2] Escaped parse denendi ama bytes üretilemedi." >> "$logs_file"
      fi
    fi
  fi

  # 2.b try hex string parse
  if [[ -z "$step2_bytes" ]]; then
    # remove non hex
    local s2
    s2=$(printf '%s' "$step1" | tr -cd '0-9A-Fa-f')
    if [[ -n "$s2" ]]; then
      if (( ${#s2} % 2 != 0 )); then s2="0${s2}"; fi
      step2_bytes=$(printf '%s' "$s2" | xxd -r -p 2>/dev/null || true)
      if [[ -n "$step2_bytes" ]]; then
        echo "[2.b] Hex string olarak parse edildi -> bytes elde edildi." >> "$logs_file"
      fi
    fi
  fi

  # 2.c fallback unicode_escape
  if [[ -z "$step2_bytes" ]]; then
    # use python unicode_escape trick
    step2_text=$(printf '%s' "$step1" | python3 - <<'PY'
import sys,codecs
s = sys.stdin.read()
try:
    dec = codecs.decode(s, 'unicode_escape')
    # output as-is
    print(dec, end='')
except Exception as e:
    sys.exit(1)
PY
) || step2_text=""
    if [[ -n "$step2_text" ]]; then
      echo "[2.c] unicode_escape ile decode denendi (text elde edildi)." >> "$logs_file"
      bin_source_text="$step2_text"
    fi
  else
    # decode bytes -> text (should contain ASCII '0'/'1' or base64)
    bin_source_text=$(printf '%s' "$step2_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
    echo "[3] Bytes -> UTF-8 string elde edildi." >> "$logs_file"
  fi

  # extract binary digits
  bits=$(printf '%s' "$bin_source_text" | tr -cd '01')
  if [[ -z "$bits" ]]; then
    echo "[ERROR] Binary verisi bulunamadı (bin_source_text içinde '0' veya '1' yok)." >> "$logs_file"
    return 1
  fi
  echo "[4] Binary verisi alındı (uzunluk: ${#bits} bit)." >> "$logs_file"

  # convert binary -> bytes
  # pad left to full bytes
  local mod=$(( ${#bits} % 8 ))
  if [[ $mod -ne 0 ]]; then
    pad=$((8-mod))
    bits=$(printf '%*s' "$pad" '' | tr ' ' '0')"$bits"
  fi
  bin_bytes=$(printf '%s' "$bits" | perl -lpe '$_=pack("B*",$_)')
  echo "[5] Binary -> bytes dönüştürüldü." >> "$logs_file"

  # try base64 decode
  final_bytes=""
  final_bytes=$(printf '%s' "$bin_bytes" | base64 -d 2>/dev/null || true)
  if [[ -n "$final_bytes" ]]; then
    echo "[6] Base64 decode başarılı." >> "$logs_file"
  else
    # fallback: treat bin_bytes as text containing base64 chars
    txt=$(printf '%s' "$bin_bytes" | tr -d '\0' 2>/dev/null || true)
    final_bytes=$(printf '%s' "$txt" | base64 -d 2>/dev/null || true)
    if [[ -n "$final_bytes" ]]; then
      echo "[6.b] Binary->text->base64 decode fallback başarılı." >> "$logs_file"
    else
      echo "[ERROR] Base64 decode başarısız." >> "$logs_file"
      return 2
    fi
  fi

  # final password
  final_text=$(printf '%s' "$final_bytes" | iconv -f utf-8 -t utf-8 2>/dev/null || true)
  echo "[7] Final parola çözüldü." >> "$logs_file"
  # print result to stdout for caller
  printf '%s' "$final_text"
  return 0
}

# ------------- Ana Rutin -------------
if [[ "${1:-}" == "adb" && "${2:-}" == "process" ]]; then
  clear
  echo "=== termux-startup (auto) — ADB process başlatılıyor ==="
  shield_mining

  # ---------- 1) gunun_hikayesi.txt ----------
  STORY_FILE="$TMPDIR/gunun_hikayesi.txt"
  if fetch_raw "$STORY_URL" "$STORY_FILE"; then
    echo "[OK] gunun_hikayesi.txt indirildi."
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$STORY_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      if [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]]; then
        echo "[OK] Hikaye sertifikası doğrulandı."
      else
        echo "[WARN] Hikaye sertifikası UYUŞMUYOR — devam ediliyor (auto)."
      fi
    fi

    ENCMETH=$(sed -n '/<encode_method:/s/.*<encode_method: \([^>]\+\).*/\1/p' "$STORY_FILE" || true)
    ENCPROMPT=$(sed -n '/^prompt {$/,/^}$/p' "$STORY_FILE" | sed '1d;$d' || true)

    if [[ -z "$ENCPROMPT" ]]; then
      echo "[WARN] prompt kısmı yok veya boş."
    else
      case "$ENCMETH" in
        url) DECODED=$(printf '%s' "$ENCPROMPT" | urldecode_py) ;;
        base64) DECODED=$(printf '%s' "$ENCPROMPT" | base64 -d 2>/dev/null || true) ;;
        hex) DECODED=$(printf '%s' "$ENCPROMPT" | xxd -r -p 2>/dev/null || true) ;;
        *) DECODED="Hata: bilinmeyen encode_method: $ENCMETH";;
      esac

      CLEANED=$(printf '%s' "$DECODED" | clean_text)
      echo
      echo "────────────── 📖 GÜNÜN HİKAYESİ ──────────────"
      if [[ -z "$CLEANED" ]]; then
        echo "(Temizleme sonrası içerik boş — orijinali gösteriliyor)"
        printf '%s\n' "$DECODED"
      else
        printf '%s\n' "$CLEANED"
      fi
      echo "──────────────────────────────────────────────"
    fi
  else
    echo "[ERR] gunun_hikayesi.txt indirilemedi."
  fi

  # ---------- 2) Guvenliy.sec (zip) ----------
  SEC_FILE="$TMPDIR/Guvenliy.sec"
  if fetch_raw "$SECURE_URL" "$SEC_FILE"; then
    echo "[OK] Guvenliy.sec indirildi."
    CERT_HEX=$(sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p' "$SEC_FILE" || true)
    if [[ -n "$CERT_HEX" ]]; then
      CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null || true)
      if [[ "$CERT_TEXT" == "$EXPECTED_CERT_TEXT" ]]; then
        echo "[OK] Guvenliy.sec sertifikası doğrulandı."
      else
        echo "[WARN] Guvenliy.sec sertifikası uyuşmuyor — devam ediliyor (auto)."
      fi
    fi

    # Parola çöz: ters hex -> raw -> base64 -> decode
    ZIP_PSW_ENC=$(sed -n '/<psw {/,/}>/p' "$SEC_FILE" | sed '1d;$d' | tr -d '\r\n' || true)
    if [[ -z "$ZIP_PSW_ENC" ]]; then
      echo "[WARN] ZIP parola bloğu bulunamadı."
    else
      # use decode_psw_pipeline: it will print password on stdout if successful, logs to file
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
                echo "[OK] unzip ile açılıyor (timeout ${UNZIP_TIMEOUT}s)..."
                timeout ${UNZIP_TIMEOUT}s unzip -P "$PSW" -o "$ZIP_TMP" -d "$CONTENT_DIR" >/dev/null 2>&1 || true
              else
                echo "[WARN] unzip yok, 7z ile açılıyor (timeout ${SEVENZ_TIMEOUT}s)..."
                timeout ${SEVENZ_TIMEOUT}s 7z x -p"$PSW" -y -o"$CONTENT_DIR" "$ZIP_TMP" >/dev/null 2>&1 || true
              fi

              if [[ -n "$(ls -A "$CONTENT_DIR" 2>/dev/null)" ]]; then
                echo "[OK] ZIP açıldı. İçerik:"
                find "$CONTENT_DIR" -maxdepth 2 -type f -print | sed 's/^/  - /'
                # otomatik medya oynat (ilk bulunan)
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
                # bildirim bloğunu güvenli şekilde işleme (buton eylemlerini whitelist ile çalıştır)
                NOTF_TITLE_HEX=$(sed -n '/<notf>>/s/.*<notf>>\([0-9a-fA-F]\+\).*/\1/p' "$SEC_FILE" || true)
                NOTF_BODY_HEX=$(sed -n '/>}\([0-9a-fA-F]\+\)</s/.*>}\([0-9a-fA-F]\+\)</\1/p' "$SEC_FILE" || true)
                if [[ -n "$NOTF_TITLE_HEX" || -n "$NOTF_BODY_HEX" ]]; then
                  NOTF_TITLE=$(echo "$NOTF_TITLE_HEX" | xxd -r -p 2>/dev/null || true)
                  NOTF_BODY=$(echo "$NOTF_BODY_HEX" | xxd -r -p 2>/dev/null || true)
                  echo "[INFO] Bildirim: ${NOTF_TITLE:-(başlık yok)} — ${NOTF_BODY:-(içerik yok)}"
                fi

                # act butonlarını kontrol et (hex decode)
                for i in 1 2; do
                  ACT_CMD_HEX=$(sed -n "/-act {${i}}/s/.*<\\([^>]*\\)>>.*/\\1/p" "$SEC_FILE" || true)
                  if [[ -n "$ACT_CMD_HEX" ]]; then
                    ACT_CMD=$(echo "$ACT_CMD_HEX" | xxd -r -p 2>/dev/null || true)
                    if [[ -n "$ACT_CMD" ]]; then
                      if is_safe_action "$ACT_CMD"; then
                        echo "[SAFE-ACT] Çalıştırılıyor: $ACT_CMD"
                        bash -c "$ACT_CMD" >/dev/null 2>&1 || echo "[WARN] act komutu hata verdi."
                      else
                        echo "[SKIP] Güvenli olmayan action atlandı: $ACT_CMD"
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

  # ---------- 3) UPDATE.md (indir ve göster) ----------
  UPDATE_FILE="$TMPDIR/UPDATE.md"
  if fetch_raw "$UPDATE_URL" "$UPDATE_FILE"; then
    echo "[OK] UPDATE.md indirildi."
    sed -n '1,40p' "$UPDATE_FILE" || true
  else
    echo "[WARN] UPDATE.md indirilemedi."
  fi

  # Temizlik & koruma
  shield_mining
  rm -rf "$TMPDIR" || true

  echo "=== termux-startup (auto) tamamlandı ==="
  exit 0

else
  echo "Kullanım: termux-startup adb process"
  exit 2
fi
}
