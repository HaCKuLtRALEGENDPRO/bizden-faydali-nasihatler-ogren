<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1760597052>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/bin/bash

# Yardımcılar: Harici bağımlılıksız (xxd/python) decoder fonksiyonları
decode_hex_to_ascii() {
    local hex="$1"
    if [ -z "$hex" ]; then
        echo "Boş hex girişi" >&2
        return 1
    fi
    case "$hex" in
        *[!0-9a-fA-F]*)
            echo "Geçersiz hex girişi" >&2
            return 1
            ;;
    esac
    if [ $(( ${#hex} % 2 )) -ne 0 ]; then
        echo "Hex uzunluğu çift olmalı" >&2
        return 1
    fi
    printf '%b' "$(echo "$hex" | sed 's/../\\x&/g')"
}

url_decode() {
    local data="${1//+/ }"
    printf '%b' "${data//%/\\x}"
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
CERT_BASE64=$(decode_hex_to_ascii "$CERT_HEX" 2>&1)
if [ $? -ne 0 ] || [ -z "$CERT_BASE64" ]; then
    echo "Hata: Sertifika HEX decode başarısız! [Bizden iyi nasihatler öğren]"
    echo "Hata detayı: $CERT_BASE64"
    exit 1
fi
CERT_TEXT=$(echo "$CERT_BASE64" | base64 -d 2>&1)
if [ $? -ne 0 ] || [ -z "$CERT_TEXT" ]; then
    echo "Hata: Sertifika Base64 decode başarısız! [Bizden iyi nasihatler öğren]"
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
TOAST_MESSAGE=$(decode_hex_to_ascii "$TOAST_HEX" 2>&1)
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
NO_TOAST_MESSAGE=$(decode_hex_to_ascii "$NO_TOAST_HEX" 2>&1)
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

# Runtime kısmını çıkar ve güncelle
RUNTIME_CONTENT=$(echo "$UPDATE_CONTENT" | sed -n '/^runtime {$/,/^}$/p' | sed '1d;$d')
if [ -z "$RUNTIME_CONTENT" ]; then
    echo "Hata: Runtime içeriği eksik! [Bizden iyi nasihatler öğren]"
    exit 1
fi
echo "$RUNTIME_CONTENT" > "$PREFIX/bin/termux-startup"
chmod +x "$PREFIX/bin/termux-startup"
echo "termux-startup güncellendi!"

# Normal hikaye işleme
if [ "$1" = "adb" ] && [ "$2" = "process" ]; then
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
    CERT_BASE64=$(decode_hex_to_ascii "$CERT_HEX" 2>&1)
    if [ $? -ne 0 ] || [ -z "$CERT_BASE64" ]; then
        echo "Hata: Sertifika HEX decode başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: $CERT_BASE64"
        exit 1
    fi
    CERT_TEXT=$(echo "$CERT_BASE64" | base64 -d 2>&1)
    if [ $? -ne 0 ] || [ -z "$CERT_TEXT" ]; then
        echo "Hata: Sertifika Base64 decode başarısız! [Bizden iyi nasihatler öğren]"
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
    TOAST_MESSAGE=$(decode_hex_to_ascii "$TOAST_HEX" 2>&1)
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
    NO_TOAST_MESSAGE=$(decode_hex_to_ascii "$NO_TOAST_HEX" 2>&1)
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
            STORY=$(url_decode "$ENCODED_STORY" 2>&1)
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
            STORY=$(decode_hex_to_ascii "$ENCODED_STORY" | tr -d '\n' 2>&1)
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
