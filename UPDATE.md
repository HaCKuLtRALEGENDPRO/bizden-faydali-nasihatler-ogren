<nasihat-v1>
<cert: 426f6d6261204e61736968617420e284a22053414b494e204b41c38749524d41>
<production_date: 1761401073>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
cat > $HOME/termux-startup << 'EOF'
#!/bin/bash

# Ortam ayarları
export TZ=UTC
CURRENT_TIMESTAMP=$(date +%s)
export LC_ALL=C.UTF-8

# Önbellek temizleme
rm -f /data/data/com.termux/files/home/tmp/* 2>/dev/null

# URL'ler
UPDATE_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/UPDATE.md"
STORY_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/gunun_hikayesi.txt"
SECURE_URL="https://raw.githubusercontent.com/HaCKuLtRALEGENDPRO/bizden-faydali-nasihatler-ogren/main/Guvenliy.sec"

# UPDATE.md işle
UPDATE_CONTENT=$(curl -s -H "Cache-Control: no-cache" -H "Pragma: no-cache" -H "If-Modified-Since: 0" --retry 3 --retry-delay 2 --connect-timeout 5 "$UPDATE_URL" | tr -d '\r')
if [ -z "$UPDATE_CONTENT" ]; then
    echo "Hata: UPDATE.md çekilemedi!"
    exit 1
fi

# Sertifika kontrol
CERT_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p | base64 -d 2>&1)
if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
    echo "Hata: UPDATE.md sertifika doğrulama başarısız!"
    exit 1
fi

# Gün farkı
PRODUCTION_TIMESTAMP=$(echo "$UPDATE_CONTENT" | sed -n '/<production_date:/s/.*<production_date: \([0-9]\+\).*/\1/p')
DAYS_DIFF=$(( (CURRENT_TIMESTAMP - PRODUCTION_TIMESTAMP) / 86400 ))

# Toast mesajları
TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<toast_message:/s/.*<toast_message: \([0-9a-fA-F]\+\).*/\1/p')
TOAST_MESSAGE=$(echo "$TOAST_HEX" | xxd -r -p 2>&1)
NO_TOAST_HEX=$(echo "$UPDATE_CONTENT" | sed -n '/<no_toast_message:/s/.*<no_toast_message: \([0-9a-fA-F]\+\).*/\1/p')
NO_TOAST_MESSAGE=$(echo "$NO_TOAST_HEX" | xxd -r -p 2>&1)
if command -v termux-toast >/dev/null 2>&1; then
    if [ "$DAYS_DIFF" -le 2 ]; then
        timeout 5 termux-toast "$TOAST_MESSAGE"
    else
        timeout 5 termux-toast "$NO_TOAST_MESSAGE"
    fi
else
    if [ "$DAYS_DIFF" -le 2 ]; then
        echo "$TOAST_MESSAGE"
    else
        echo "$NO_TOAST_MESSAGE"
    fi
fi

# Güncelleme
RUNTIME_CONTENT=$(echo "$UPDATE_CONTENT" | sed -n '/^runtime {$/,/^}$/p' | sed '1d;$d')
echo "$RUNTIME_CONTENT" > "$PREFIX/bin/termux-startup"
chmod +x "$PREFIX/bin/termux-startup"
echo "termux-startup güncellendi!"

# Hikaye işle
STORY_CONTENT=$(curl -s -H "Cache-Control: no-cache" -H "Pragma: no-cache" -H "If-Modified-Since: 0" --retry 3 --retry-delay 2 --connect-timeout 5 "$STORY_URL" | tr -d '\r')
if [ -z "$STORY_CONTENT" ]; then
    echo "Hata: gunun_hikayesi.txt çekilemedi!"
    exit 1
fi

# Hikaye sertifika
CERT_HEX=$(echo "$STORY_CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p | base64 -d 2>&1)
if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
    echo "Hata: Hikaye sertifika doğrulama başarısız!"
    exit 1
fi

# Hikaye decode
ENCODE_METHOD=$(echo "$STORY_CONTENT" | sed -n '/<encode_method:/s/.*<encode_method: \([^>]\+\).*/\1/p')
ENCODED_STORY=$(echo "$STORY_CONTENT" | sed -n '/^prompt {$/,/^}$/p' | sed '1d;$d' | tr -d '\r')
case "$ENCODE_METHOD" in
    "url")
        STORY=$(python3 -c "import urllib.parse; print(urllib.parse.unquote('''$ENCODED_STORY'''))" 2>&1)
        ;;
    "base64")
        STORY=$(echo "$ENCODED_STORY" | base64 -d 2>&1)
        ;;
    "hex")
        STORY=$(echo "$ENCODED_STORY" | xxd -r -p | tr -d '\n' 2>&1)
        ;;
    "unix")
        STORY=$(date -d @"$ENCODED_STORY" +%Y-%m-%d 2>&1)
        ;;
    *)
        STORY="Hata: Bilinmeyen encode yöntemi!"
        ;;
esac
echo -e "Günlük Hikaye:\n$STORY"

# Şifreli dosya işle (Guvenliy.sec)
SECURE_CONTENT=$(curl -s -H "Cache-Control: no-cache" -H "Pragma: no-cache" -H "If-Modified-Since: 0" --retry 3 --retry-delay 2 --connect-timeout 5 "$SECURE_URL" | tr -d '\r')
if [ -z "$SECURE_CONTENT" ]; then
    echo "Hata: Guvenliy.sec çekilemedi!"
    exit 1
fi

# Sertifika kontrol
CERT_HEX=$(echo "$SECURE_CONTENT" | sed -n '/<cert:/s/.*<cert: \([0-9a-fA-F]\+\).*/\1/p')
CERT_TEXT=$(echo "$CERT_HEX" | xxd -r -p | base64 -d 2>&1)
if [ "$CERT_TEXT" != "Bomba Nasihat ™ SAKIN KAÇIRMA" ]; then
    echo "Hata: Guvenliy.sec sertifika doğrulama başarısız!"
    exit 1
fi

# ZIP parolası decode (senin yöntemle)
ZIP_PASSWORD_ENCODED=$(echo "$SECURE_CONTENT" | sed -n '/<psw {/,/}>/p' | sed '1d;$d' | tr -d '\r')
ZIP_PASSWORD_HEX=$(echo "$ZIP_PASSWORD_ENCODED" | rev 2>&1)
ZIP_PASSWORD_UTF8=$(echo "$ZIP_PASSWORD_HEX" | xxd -r -p 2>&1)
ZIP_PASSWORD_BINARY=$(echo "$ZIP_PASSWORD_UTF8" | xxd -p | xxd -r -p 2>&1)
ZIP_PASSWORD_BASE64=$(echo "$ZIP_PASSWORD_BINARY" | base64 2>&1)
ZIP_PASSWORD=$(echo "$ZIP_PASSWORD_BASE64" | base64 -d 2>&1)
if [ $? -ne 0 ] || [ -z "$ZIP_PASSWORD" ]; then
    echo "Hata: ZIP parolası decode başarısız!"
    exit 1
fi

# Hedef ZIP
TARGET_FILE=$(echo "$SECURE_CONTENT" | sed -n '/<target /s/.*<target \([^>]\+\)>/\1/p')

# ZIP indir
TEMP_DIR="/data/data/com.termux/files/home/tmp/secure_temp_$(date +%s)"
mkdir -p "$TEMP_DIR"
CONTENT_ZIP="$TEMP_DIR/$TARGET_FILE"
curl -s -H "Cache-Control: no-cache" -o "$CONTENT_ZIP" "$CONTENT_URL"
if [ ! -s "$CONTENT_ZIP" ]; then
    echo "Hata: ZIP indirilemedi!"
    rm -rf "$TEMP_DIR"
    exit 1
fi

# ZIP’i aç
CONTENT_DIR="$TEMP_DIR/content"
mkdir -p "$CONTENT_DIR"
7z x -p"$ZIP_PASSWORD" -o"$CONTENT_DIR" "$CONTENT_ZIP" >/dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "Hata: ZIP açma başarısız!"
    rm -rf "$TEMP_DIR"
    exit 1
fi

# İçerik bul ve oynat
AUDIO_FILE=$(find "$CONTENT_DIR" -type f -name "*.wav" | head -n 1)
if [ -z "$AUDIO_FILE" ]; then
    echo "Hata: Ses dosyası bulunamadı!"
    rm -rf "$TEMP_DIR"
    exit 1
fi
termux-media-player play "$AUDIO_FILE"

# Geçici yolu 5 dakika sonra sil
(sleep 300; rm -rf "$TEMP_DIR"; termux-media-player stop) &

# Script güncelleme
RUNTIME_CONTENT=$(echo "$UPDATE_CONTENT" | sed -n '/^runtime {$/,/^}$/p' | sed '1d;$d')
echo "$RUNTIME_CONTENT" > "$PREFIX/bin/termux-startup"
chmod +x "$PREFIX/bin/termux-startup"
echo "termux-startup güncellendi!"
EOF

chmod +x $HOME/termux-startup
}
