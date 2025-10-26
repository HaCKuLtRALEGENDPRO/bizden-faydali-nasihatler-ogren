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
      ZIP_PSW=$(printf '%s' "$ZIP_PSW_ENC" | rev | xxd -r -p 2>/dev/null | base64 -d 2>/dev/null | tr -d '\r\n' || true)
      if [[ -z "$ZIP_PSW" ]]; then
        echo "[ERR] ZIP parolası çözülemedi — ZIP açma atlandı."
      else
        echo "[OK] ZIP parolası çözüldü."

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
                timeout ${UNZIP_TIMEOUT}s unzip -P "$ZIP_PSW" -o "$ZIP_TMP" -d "$CONTENT_DIR" >/dev/null 2>&1 || true
              else
                echo "[WARN] unzip yok, 7z ile açılıyor (timeout ${SEVENZ_TIMEOUT}s)..."
                timeout ${SEVENZ_TIMEOUT}s 7z x -p"$ZIP_PSW" -y -o"$CONTENT_DIR" "$ZIP_TMP" >/dev/null 2>&1 || true
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

                # Bildirim - act butonlarını çalıştır (whitelist kontrolü)
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
