<nasihat-v1>
<cert: 516D3974596D4567546D467A6157686864434469684B49675530464C535534675330484468306C535455453D>
<production_date: 1761247877>
<toast_message: 42C39C46452041C387494C4449204B41524445C59E494D21>
<no_toast_message: 53414B494E2047C39C4E44454DC4B0204B41C38749524D41>
runtime {
#!/data/data/com.termux/files/usr/bin/bash
# 🧠 Termux Startup Command — 717 Edition
# Kullanım: termux-startup adb process

arg1=$1
arg2=$2

# Sadece "adb process" parametresiyle çalışsın
if [[ "$arg1" == "adb" && "$arg2" == "process" ]]; then
  clear
  echo "────────────────────────────────────────────"
  echo "🧩 717 Termux-Startup: ADB Process Başlatılıyor..."
  echo "────────────────────────────────────────────"
  sleep 1

  # 1️⃣ GÜNÜN HİKAYESİ
  story_file="$HOME/gunun_hikayesi.txt"
  if [ -f "$story_file" ]; then
    echo "📜 Günün Hikayesi yükleniyor..."
    sleep 1

    # Hikaye metnini oku ve ters karakterleri düzelt
    clean_story=$(awk '
    {
      # Uzun veya bozuk satırları atla
      if (length($0) > 150 || gsub(/[A-Za-z0-9]/,"&") < (length($0)/3)) next
      # Satır tersse düzelt
      cmd = "echo " $0 " | rev"
      cmd | getline fixed
      close(cmd)
      if (fixed ~ /^[[:print:]]+$/) print fixed; else print $0
    }' "$story_file")

    echo ""
    echo "🌀 HİKAYE BAŞLIYOR 🌀"
    echo "────────────────────────────────────────────"
    echo "$clean_story"
    echo "────────────────────────────────────────────"
  else
    echo "[!] gunun_hikayesi.txt bulunamadı."
  fi

  # 2️⃣ GÜVENLİ DOSYA AÇMA (Guvenliy.sec)
  SECFILE="$HOME/Guvenliy.sec"
  DESTDIR="$HOME/.guvenli_dosya"
  mkdir -p "$DESTDIR"

  if [ -f "$SECFILE" ]; then
    echo ""
    echo "🔐 Guvenliy.sec tespit edildi, çözülüyor..."
    file_type=$(file "$SECFILE")

    if echo "$file_type" | grep -qi "zip"; then
      unzip -oq -P "717" "$SECFILE" -d "$DESTDIR"
    elif echo "$file_type" | grep -qi "gzip"; then
      tar -xzf "$SECFILE" -C "$DESTDIR"
    fi

    if [ "$(ls -A $DESTDIR 2>/dev/null)" ]; then
      echo "✅ Guvenli içerik başarıyla açıldı."
    else
      echo "⚠️  Dosya açılmadı veya parola hatalı."
    fi
  else
    echo "[!] Guvenliy.sec bulunamadı."
  fi

  # 3️⃣ UPDATE.md kontrolü
  update_file="$HOME/UPDATE.md"
  if [ -f "$update_file" ]; then
    echo ""
    echo "🧩 UPDATE.md bulundu — güncelleme notları:"
    echo "────────────────────────────────────────────"
    head -n 10 "$update_file"
    echo "────────────────────────────────────────────"
  fi

  # 4️⃣ Stabilizasyon ve toast
  termux-toast -b black -c green "717 Startup Process tamamlandı!"
  echo ""
  echo "✨ Sistem stabilize edildi. Hoş geldin 717 👑"
  echo "────────────────────────────────────────────"
else
  echo "Kullanım: termux-startup adb process"
fi
}
