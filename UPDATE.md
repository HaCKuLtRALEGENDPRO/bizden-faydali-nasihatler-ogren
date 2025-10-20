<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1760597052>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/bin/bash

set -euo pipefail

# Güvenli HEX → metin dönüştürücü (xxd yoksa python3 ile fallback)
hex_to_text() {
    local hex_input="$1"
    if command -v xxd >/dev/null 2>&1; then
        printf '%s' "$hex_input" | xxd -r -p
        return $?
    fi
    if command -v python3 >/dev/null 2>&1; then
        python3 - "$hex_input" <<'PY'
import sys, binascii
hex_str = sys.stdin.read().strip()
try:
    sys.stdout.write(binascii.unhexlify(hex_str).decode("utf-8", "replace"))
except Exception as e:
    sys.stderr.write(str(e))
    sys.exit(1)
PY
        return $?
    fi
    echo "xxd veya python3 bulunamadı" >&2
    return 127
}

# Sertifika HEX'ini güvenle çözer; gerekirse Base64 de çözer
decode_cert_from_hex() {
    local hex_input="$1"
    local text
    text=$(hex_to_text "$hex_input" 2>/dev/null) || return 1
    # Base64 gibi görünüyorsa decode etmeyi dene; başarısızsa orijinali döndür
    if command -v base64 >/dev/null 2>&1; then
        local maybe
        maybe=$(printf '%s' "$text" | base64 -d 2>/dev/null || true)
        if [ -n "$maybe" ]; then
            printf '%s' "$maybe"
            return 0
        fi
    fi
    printf '%s' "$text"
}

# Sunucudan dosyaları çekecek URL'ler
UPDATE_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/UPDATE.md"
STORY_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/gunun_hikayesi.txt"

# Güncel tarih Unix timestamp (UTC)
export TZ=UTC
CURRENT_TIMESTAMP=$(date +%s)

# UTF-8 locale ayarını zorla
export LC_ALL=C.UTF-8

# Önbelleği temizle
rm -f /data/data/com.termux/files/home/tmp/*_cache 2>/dev/null

# UPDATE.md'yi kontrol et ve doğrula
UPDATE_CONTENT=$(curl -s -H "Cache-Control: no-cache" -H "Pragma: no-cache" -H "If-Modified-Since: 0" --retry 3 --retry-delay 2 --connect-timeout 5 "$UPDATE_URL" | tr -d '\r')
if [ -z "$UPDATE_CONTENT" ]; then
    echo "Hata: UPDATE.md çekilemedi! İnternet bağlantınızı kontrol edin. [Bizden iyi nasihatler öğren]"
    exit 1
fi

# Tanıtım string'lerini kontrol et
for tag in "<nasihat-v1>" "<cert:" "<production_date:" "<toast_message:" "<no_toast_message:" "runtime {" "}"; do
    if ! echo "$UPDATE_CONTENT" | grep -q "^$tag"; then
        echo "Hata: UPDATE.md'de '$tag' eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
done

# Sertifikayı çıkar ve decode et
CERT_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
if [ -z "$CERT_HEX" ]; then
    echo "Hata: Sertifika HEX eksik! [Bizden iyi nasihatler öğren]"
    exit 1
fi
CERT_TEXT=$(decode_cert_from_hex "$CERT_HEX" 2>&1)
if [ $? -ne 0 ] || [ -z "$CERT_TEXT" ]; then
    echo "Hata: Sertifika HEX decode başarısız! [Bizden iyi nasihatler öğren]"
    echo "Hata detayı: $CERT_TEXT"
    exit 1
fi

# Sertifika doğrulama
if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
    echo "Hata: UPDATE.md sertifika doğrulama başarısız! [Bizden iyi nasihatler öğren]"
    echo "Hata detayı: Sertifika metni: '$CERT_TEXT'"
    exit 1
fi

# Üretim tarihini çıkar
PRODUCTION_TIMESTAMP=$(echo "$UPDATE_CONTENT" | sed -n '/<production_date:/s/.*<production_date: \([0-9]\+\).*/\1/p')
if [ -z "$PRODUCTION_TIMESTAMP" ]; then
    echo "Hata: Üretim tarihi geçersiz veya eksik! [Bizden iyi nasihatler öğren]"
    exit 1
fi

# Gün farkını hesapla
DAYS_DIFF=$(( (CURRENT_TIMESTAMP - PRODUCTION_TIMESTAMP) / 86400 ))
if [ "$DAYS_DIFF" -lt 0 ]; then
    echo "Hata: Üretim tarihi gelecekte! [Bizden iyi nasihatler öğren]"
    exit 1
fi

# Toast mesajlarını çıkar ve decode et
TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<toast_message:/s/.*<toast_message: \([0-9a-fA-F]\+\).*/\1/p')
if [ -z "$TOAST_HEX" ]; then
    echo "Hata: Toast mesajı HEX eksik! [Bizden iyi nasihatler öğren]"
    exit 1
fi
TOAST_MESSAGE=$(echo "$TOAST_HEX" | hex_to_text 2>&1)
if [ $? -ne 0 ] || [ -z "$TOAST_MESSAGE" ]; then
    echo "Hata: Toast mesajı decode başarısız! [Bizden iyi nasihatler öğren]"
    echo "Hata detayı: $TOAST_MESSAGE"
    exit 1
fi
NO_TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<no_toast_message:/s/.*<no_toast_message: \([0-9a-fA-F]\+\).*/\1/p')
if [ -z "$NO_TOAST_HEX" ]; then
    echo "Hata: No toast mesajı HEX eksik! [Bizden iyi nasihatler öğren]"
    exit 1
fi
NO_TOAST_MESSAGE=$(echo "$NO_TOAST_HEX" | hex_to_text 2>&1)
if [ $? -ne 0 ] || [ -z "$NO_TOAST_MESSAGE" ]; then
    echo "Hata: No toast mesajı decode başarısız! [Bizden iyi nasihatler öğren]"
    echo "Hata detayı: $NO_TOAST_MESSAGE"
    exit 1
fi

# Toast göster
if ! command -v termux-toast >/dev/null 2>&1; then
    if [ "$DAYS_DIFF" -le 2 ]; then
        echo "$TOAST_MESSAGE"
    else
        echo "$NO_TOAST_MESSAGE"
    fi
else
    if [ "$DAYS_DIFF" -le 2 ]; then
        timeout 5 termux-toast "$TOAST_MESSAGE" || echo "Uyarı: Toast gösterilemedi, hikaye devam ediyor."
    else
        timeout 5 termux-toast "$NO_TOAST_MESSAGE" || echo "Uyarı: No toast gösterilemedi, hikaye devam ediyor."
    fi
fi

# Runtime kısmını çıkar (iç içe süslü parantezleri doğru eşle)
RUNTIME_CONTENT=$(printf '%s\n' "$UPDATE_CONTENT" | python3 - <<'PY'
import sys
data = sys.stdin.read().replace('\r','')
lines = data.splitlines()
in_block = False
depth = 0
out = []
for line in lines:
    if not in_block:
        if line.strip() == 'runtime {':
            in_block = True
            depth = 1
        continue
    else:
        # Güncel satırdaki süslüleri say
        open_count = line.count('{')
        close_count = line.count('}')
        depth += open_count - close_count
        # depth < 1 olduğunda dıştaki runtime kapanmış demektir; bu satırı dahil etme
        if depth < 1:
            break
        out.append(line)
print('\n'.join(out))
PY
)
if [ -z "$RUNTIME_CONTENT" ]; then
    echo "Hata: Runtime içeriği parse edilemedi! [Bizden iyi nasihatler öğren]"
    exit 1
fi
printf '%s\n' "$RUNTIME_CONTENT" > "$PREFIX/bin/termux-startup"
chmod 0755 "$PREFIX/bin/termux-startup"
echo "termux-startup güncellendi!"

# Normal hikaye işleme
if [ "${1-}" = "adb" ] && [ "${2-}" = "process" ]; then
    CONTENT=$(curl -s -H "Cache-Control: no-cache" -H "Pragma: no-cache" --retry 3 --retry-delay 2 --connect-timeout 5 "$STORY_URL" | tr -d '\r')
    if [ -z "$CONTENT" ]; then
        echo "Hata: Dosya çekilemedi! İnternet bağlantınızı kontrol edin. [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Tanıtım string'lerini kontrol et
    for tag in "<nasihat-v1>" "<cert:" "<production_date:" "<toast_message:" "<no_toast_message:" "<encode_method:" "prompt {" "}"; do
        if ! echo "$CONTENT" | grep -q "^$tag"; then
            echo "Hata: gunun_hikayesi.txt'de '$tag' eksik! [Bizden iyi nasihatler öğren]"
            exit 1
        fi
    done

    # Sertifikayı çıkar ve decode et
    CERT_HEX=$(echo "$CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
    if [ -z "$CERT_HEX" ]; then
        echo "Hata: Sertifika HEX eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    CERT_TEXT=$(decode_cert_from_hex "$CERT_HEX" 2>&1)
    if [ $? -ne 0 ] || [ -z "$CERT_TEXT" ]; then
        echo "Hata: Sertifika HEX decode başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: $CERT_TEXT"
        exit 1
    fi

    # Sertifika doğrulama
    if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
        echo "Hata: Sertifika doğrulama başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: Sertifika metni: '$CERT_TEXT'"
        exit 1
    fi

    # Üretim tarihini çıkar
    PRODUCTION_TIMESTAMP=$(echo "$CONTENT" | sed -n '/<production_date:/s/.*<production_date: \([0-9]\+\).*/\1/p')
    if [ -z "$PRODUCTION_TIMESTAMP" ]; then
        echo "Hata: Üretim tarihi geçersiz veya eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Gün farkını hesapla
    DAYS_DIFF=$(( (CURRENT_TIMESTAMP - PRODUCTION_TIMESTAMP) / 86400 ))
    if [ "$DAYS_DIFF" -lt 0 ]; then
        echo "Hata: Üretim tarihi gelecekte! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Toast mesajlarını çıkar ve decode et
    TOAST_HEX=$(echo "$CONTENT" | sed -n '/<toast_message:/s/.*<toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    if [ -z "$TOAST_HEX" ]; then
        echo "Hata: Toast mesajı HEX eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    TOAST_MESSAGE=$(echo "$TOAST_HEX" | hex_to_text 2>&1)
    if [ $? -ne 0 ] || [ -z "$TOAST_MESSAGE" ]; then
        echo "Hata: Toast mesajı decode başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: $TOAST_MESSAGE"
        exit 1
    fi
    NO_TOAST_HEX=$(echo "$CONTENT" | sed -n '/<no_toast_message:/s/.*<no_toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    if [ -z "$NO_TOAST_HEX" ]; then
        echo "Hata: No toast mesajı HEX eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    NO_TOAST_MESSAGE=$(echo "$NO_TOAST_HEX" | hex_to_text 2>&1)
    if [ $? -ne 0 ] || [ -z "$NO_TOAST_MESSAGE" ]; then
        echo "Hata: No toast mesajı decode başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: $NO_TOAST_MESSAGE"
        exit 1
    fi

    # Toast göster
    if ! command -v termux-toast >/dev/null 2>&1; then
        if [ "$DAYS_DIFF" -le 2 ]; then
            echo "$TOAST_MESSAGE"
        else
            echo "$NO_TOAST_MESSAGE"
        fi
    else
        if [ "$DAYS_DIFF" -le 2 ]; then
            timeout 5 termux-toast "$TOAST_MESSAGE" || echo "Uyarı: Toast gösterilemedi, hikaye devam ediyor."
        else
            timeout 5 termux-toast "$NO_TOAST_MESSAGE" || echo "Uyarı: No toast gösterilemedi, hikaye devam ediyor."
        fi
    fi

    # Encode yöntemini çıkar
    ENCODE_METHOD=$(echo "$CONTENT" | sed -n '/<encode_method:/s/.*<encode_method: \([^>]\+\).*/\1/p')
    if [ -z "$ENCODE_METHOD" ]; then
        echo "Hata: Encode yöntemi eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Hikaye kısmını çıkar
    ENCODED_STORY=$(echo "$CONTENT" | sed -n '/^prompt {$/,/^}$/p' | sed '1d;$d' | tr -d '\r')
    if [ -z "$ENCODED_STORY" ]; then
        echo "Hata: Hikaye içeriği eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Hikayeyi decode et
    case "$ENCODE_METHOD" in
        "url")
            STORY=$(python3 -c "import sys, urllib.parse; print(urllib.parse.unquote('''$ENCODED_STORY'''), end='')" 2>&1)
            if [ $? -ne 0 ] || [ -z "$STORY" ]; then
                echo "Hata: URL decode başarısız! [Bizden iyi nasihatler öğren]"
                echo "Hata detayı: $STORY"
                exit 1
            fi
            ;;
        "base64")
            STORY=$(echo "$ENCODED_STORY" | base64 -d 2>&1)
            if [ $? -ne 0 ] || [ -z "$STORY" ]; then
                echo "Hata: Base64 decode başarısız! [Bizden iyi nasihatler öğren]"
                echo "Hata detayı: $STORY"
                exit 1
            fi
            ;;
        "hex")
            STORY=$(hex_to_text "$ENCODED_STORY" 2>&1 | tr -d '\n')
            if [ $? -ne 0 ] || [ -z "$STORY" ]; then
                echo "Hata: HEX decode başarısız! [Bizden iyi nasihatler öğren]"
                echo "Hata detayı: $STORY"
                exit 1
            fi
            ;;
        "unix")
            STORY=$(date -d @"$ENCODED_STORY" +%Y-%m-%d 2>&1)
            if [ $? -ne 0 ] || [ -z "$STORY" ]; then
                echo "Hata: Unix decode başarısız! [Bizden iyi nasihatler öğren]"
                echo "Hata detayı: $STORY"
                exit 1
            fi
            ;;
        *)
            echo "Hata: Bilinmeyen encode yöntemi: $ENCODE_METHOD [Bizden iyi nasihatler öğren]"
            exit 1
            ;;
    esac

    # Hikayeyi göster
    echo -e "$STORY"
fi
}
