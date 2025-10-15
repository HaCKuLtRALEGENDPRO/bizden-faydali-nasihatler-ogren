<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1759795200>
<toast_message: 42c39c46454dc4b05a494c4449204b41524445c59e>
<no_toast_message: 53414b494e2047c39c4e44454dc4b0204b41c38749524d41>
<encode_method: none>
runtime {
#!/bin/bash

# Debug için satır numarası takibi
PS4='Line ${LINENO}: '

# Sunucudan dosyaları çekecek URL'ler
UPDATE_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/UPDATE.md"
STORY_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/gunun_hikayesi.txt"

# Güncel tarih Unix timestamp (UTC)
export TZ=UTC
CURRENT_TIMESTAMP=$(date +%s)
echo "Debug: TIMEZONE=$(date +%Z)"

# UTF-8 locale ayarını zorla
export LC_ALL=C.UTF-8

# UPDATE.md'yi kontrol et ve doğrula
UPDATE_CONTENT=$(curl -s "$UPDATE_URL" | tr -d '\r')
if [ -n "$UPDATE_CONTENT" ]; then
    if ! echo "$UPDATE_CONTENT" | grep -q '^<nasihat-v1>'; then
        echo "Hata: UPDATE.md tanıtım string'i eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '<cert:'; then
        echo "Hata: UPDATE.md sertifika eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '<production_date:'; then
        echo "Hata: UPDATE.md üretim tarihi eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '<toast_message:'; then
        echo "Hata: UPDATE.md toast mesajı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '<no_toast_message:'; then
        echo "Hata: UPDATE.md no toast mesajı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '<encode_method:'; then
        echo "Hata: UPDATE.md encode yöntemi eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '^runtime {'; then
        echo "Hata: UPDATE.md runtime başı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$UPDATE_CONTENT" | grep -q '^}$'; then
        echo "Hata: UPDATE.md runtime sonu eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Sertifikayı çıkar ve decode et
    CERT_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
    echo "Debug: CERT_HEX=$CERT_HEX"
    CERT_BASE64=$(python3 -c "print(bytes.fromhex('$CERT_HEX').decode('utf-8'))" 2>/dev/null || echo "base64_decode_error")
    echo "Debug: CERT_BASE64=$CERT_BASE64"
    CERT_TEXT=$(python3 -c "import base64; print(base64.b64decode('$CERT_BASE64').decode('utf-8'))" 2>/dev/null || echo "text_decode_error")
    echo "Debug: CERT_TEXT=$CERT_TEXT"

    # Sertifika doğrulama
    if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
        echo "Hata: UPDATE.md sertifika doğrulama başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: Sertifika metni: '$CERT_TEXT'"
        exit 1
    fi

    # Üretim tarihini çıkar
    PRODUCTION_TIMESTAMP=$(echo "$UPDATE_CONTENT" | sed -n '/<production_date:/s/.*<production_date: \([0-9]\+\).*/\1/p')

    # Üretim tarihi kontrolü
    if [ -z "$PRODUCTION_TIMESTAMP" ]; then
        echo "Hata: Üretim tarihi geçersiz veya eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Gün farkını hesapla
    DAYS_DIFF=$(( (CURRENT_TIMESTAMP - PRODUCTION_TIMESTAMP) / 86400 ))
    echo "Debug: CURRENT_TIMESTAMP=$CURRENT_TIMESTAMP"
    echo "Debug: PRODUCTION_TIMESTAMP=$PRODUCTION_TIMESTAMP"
    echo "Debug: DAYS_DIFF=$DAYS_DIFF"

    # Toast mesajını HEX'ten decode et
    TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<toast_message:/s/.*<toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    TOAST_MESSAGE=$(echo "$TOAST_HEX" | xxd -r -p 2>/dev/null || echo "toast_decode_error")
    echo "Debug: TOAST_MESSAGE=$TOAST_MESSAGE"

    # No toast mesajını HEX'ten decode et
    NO_TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<no_toast_message:/s/.*<no_toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    NO_TOAST_MESSAGE=$(echo "$NO_TOAST_HEX" | xxd -r -p 2>/dev/null || echo "no_toast_decode_error")
    echo "Debug: NO_TOAST_MESSAGE=$NO_TOAST_MESSAGE"

    # Toast yerine echo eğer termux-toast yoksa
    if ! command -v termux-toast >/dev/null 2>&1; then
        if [ "$DAYS_DIFF" -le 2 ] && [ "$DAYS_DIFF" -ge 0 ]; then
            echo "Toast: $TOAST_MESSAGE"
        else
            echo "Toast: $NO_TOAST_MESSAGE"
        fi
    else
        if [ "$DAYS_DIFF" -le 2 ] && [ "$DAYS_DIFF" -ge 0 ]; then
            timeout 5 termux-toast "$TOAST_MESSAGE" || echo "Uyarı: Toast gösterilemedi, hikaye devam ediyor."
        else
            timeout 5 termux-toast "$NO_TOAST_MESSAGE" || echo "Uyarı: No toast gösterilemedi, hikaye devam ediyor."
        fi
    fi

    # Runtime kısmını çıkar
    RUNTIME_CONTENT=$(echo "$UPDATE_CONTENT" | sed -n '/^runtime {$/,/^}$/p' | sed '1d;$d')

    # termux-startup'ı tamamen yeni runtime içeriğiyle değiştir
    echo "$RUNTIME_CONTENT" > "$PREFIX/bin/termux-startup"
    chmod +x "$PREFIX/bin/termux-startup"
    echo "termux-startup güncellendi!"
fi

# Normal hikaye işleme
if [ "$1" = "adb" ] && [ "$2" = "process" ]; then
    # Sunucudan dosyayı çek
    CONTENT=$(curl -s "$STORY_URL" | tr -d '\r')

    # Dosya boşsa hata
    if [ -z "$CONTENT" ]; then
        echo "Hata: Dosya çekilemedi! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Tanıtım string’lerini kontrol et
    if ! echo "$CONTENT" | grep -q '^<nasihat-v1>'; then
        echo "Hata: Tanıtım string'i eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '<cert:'; then
        echo "Hata: Sertifika eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '<production_date:'; then
        echo "Hata: Üretim tarihi eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '<toast_message:'; then
        echo "Hata: Toast mesajı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '<no_toast_message:'; then
        echo "Hata: No toast mesajı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '<encode_method:'; then
        echo "Hata: Encode yöntemi eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '^prompt {'; then
        echo "Hata: Prompt başı eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    if ! echo "$CONTENT" | grep -q '^}$'; then
        echo "Hata: Prompt sonu eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Sertifikayı çıkar ve decode et
    CERT_HEX=$(echo "$CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
    echo "Debug: CERT_HEX=$CERT_HEX"
    CERT_BASE64=$(python3 -c "print(bytes.fromhex('$CERT_HEX').decode('utf-8'))" 2>/dev/null || echo "base64_decode_error")
    echo "Debug: CERT_BASE64=$CERT_BASE64"
    CERT_TEXT=$(python3 -c "import base64; print(base64.b64decode('$CERT_BASE64').decode('utf-8'))" 2>/dev/null || echo "text_decode_error")
    echo "Debug: CERT_TEXT=$CERT_TEXT"

    # Sertifika doğrulama
    if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
        echo "Hata: Sertifika doğrulama başarısız! [Bizden iyi nasihatler öğren]"
        echo "Hata detayı: Sertifika metni: '$CERT_TEXT'"
        exit 1
    fi

    # Üretim tarihini çıkar
    PRODUCTION_TIMESTAMP=$(echo "$CONTENT" | sed -n '/<production_date:/s/.*<production_date: \([0-9]\+\).*/\1/p')

    # Üretim tarihi kontrolü
    if [ -z "$PRODUCTION_TIMESTAMP" ]; then
        echo "Hata: Üretim tarihi geçersiz veya eksik! [Bizden iyi nasihatler öğren]"
        exit 1
    fi

    # Gün farkını hesapla
    DAYS_DIFF=$(( (CURRENT_TIMESTAMP - PRODUCTION_TIMESTAMP) / 86400 ))
    echo "Debug: CURRENT_TIMESTAMP=$CURRENT_TIMESTAMP"
    echo "Debug: PRODUCTION_TIMESTAMP=$PRODUCTION_TIMESTAMP"
    echo "Debug: DAYS_DIFF=$DAYS_DIFF"

    # Toast mesajını HEX'ten decode et
    TOAST_HEX=$(echo "$CONTENT" | sed -n '/<toast_message:/s/.*<toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    TOAST_MESSAGE=$(echo "$TOAST_HEX" | xxd -r -p 2>/dev/null || echo "toast_decode_error")
    echo "Debug: TOAST_MESSAGE=$TOAST_MESSAGE"

    # No toast mesajını HEX'ten decode et
    NO_TOAST_HEX=$(echo "$CONTENT" | sed -n '/<no_toast_message:/s/.*<no_toast_message: \([0-9a-fA-F]\+\).*/\1/p')
    NO_TOAST_MESSAGE=$(echo "$NO_TOAST_HEX" | xxd -r -p 2>/dev/null || echo "no_toast_decode_error")
    echo "Debug: NO_TOAST_MESSAGE=$NO_TOAST_MESSAGE"

    # Toast yerine echo eğer termux-toast yoksa
    if ! command -v termux-toast >/dev/null 2>&1; then
        if [ "$DAYS_DIFF" -le 2 ] && [ "$DAYS_DIFF" -ge 0 ]; then
            echo "Toast: $TOAST_MESSAGE"
        else
            echo "Toast: $NO_TOAST_MESSAGE"
        fi
    else
        if [ "$DAYS_DIFF" -le 2 ] && [ "$DAYS_DIFF" -ge 0 ]; then
            timeout 5 termux-toast "$TOAST_MESSAGE" || echo "Uyarı: Toast gösterilemedi, hikaye devam ediyor."
        else
            timeout 5 termux-toast "$NO_TOAST_MESSAGE" || echo "Uyarı: No toast gösterilemedi, hikaye devam ediyor."
        fi
    fi

    # Encode yöntemini çıkar
    ENCODE_METHOD=$(echo "$CONTENT" | sed -n '/<encode_method:/s/.*<encode_method: \([^>]\+\).*/\1/p')
    echo "Debug: ENCODE_METHOD=$ENCODE_METHOD"

    # Hikaye kısmını çıkar
    ENCODED_STORY=$(echo "$CONTENT" | sed -n '/^prompt {$/,/^}$/p' | sed '1d;$d' | tr -d '\r')
    echo "Debug: ENCODED_STORY=$ENCODED_STORY"

    # Hikayeyi decode et
    if [ "$ENCODE_METHOD" = "url" ]; then
        STORY=$(python3 -c "import urllib.parse; print(urllib.parse.unquote('''$ENCODED_STORY'''))" 2>/dev/null || echo "url_decode_error")
    elif [ "$ENCODE_METHOD" = "base64" ]; then
        STORY=$(echo "$ENCODED_STORY" | base64 -d 2>/dev/null || echo "base64_decode_error")
    elif [ "$ENCODE_METHOD" = "hex" ]; then
        STORY=$(echo "$ENCODED_STORY" | xxd -r -p | tr -d '\n' 2>/dev/null || echo "hex_decode_error")
    elif [ "$ENCODE_METHOD" = "unix" ]; then
        STORY=$(date -d @"$ENCODED_STORY" +%Y-%m-%d 2>/dev/null || echo "unix_decode_error")
    else
        echo "Hata: Bilinmeyen encode yöntemi! [Bizden iyi nasihatler öğren]"
        exit 1
    fi
    echo "Debug: STORY=$STORY"

    # Hikayeyi göster
    echo -e "$STORY"
fi
}
