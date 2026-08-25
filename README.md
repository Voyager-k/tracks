#debian
sudo apt update
sudo apt install mpv curl gawk grep sed

#arch
sudo pacman -S mpv curl awk grep sed

#after edit nano
chmod +x ~/.local/bin/play-gh

echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
________________________________________________________

mkdir -p ~/.local/bin
nano ~/.local/bin/play-gh

#!/bin/bash

REPO="Voyager-k/tracks"
BRANCH="main"

# Fetch songs and properly URL-encode the download links using python3 (safe for #, spaces, etc.)
FETCH_SONGS() {
    curl -s "https://api.github.com/repos/$REPO/contents?ref=$BRANCH" | \
    grep -o '"download_url": *"[^"]*"' | \
    cut -d'"' -f4 | \
    grep -E '\.(mp3|flac|wav|m4a|ogg)$' | \
    while read -r url; do
        # Clean filename for display
        filename=$(basename "$url" | sed 's/%20/ /g' | sed 's/\.[^.]*$//')
        # URL-encode the raw link safely so # and spaces don't break curl/mpv
        encoded_url=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$url', safe=':/%'))")
        echo "$filename|$encoded_url"
    done
}

# If no arguments are provided, list all available tracks with numbers
if [ $# -eq 0 ]; then
    echo "Available tracks in $REPO:"
    echo "----------------------------------------"
    FETCH_SONGS | awk -F'|' '{print "  " NR ") " $1}'
    echo "----------------------------------------"
    echo "Usage: play-gh <number> [number2 number3 ...]"
    echo "Example: play-gh 1 3 5"
    exit 0
fi

PLAYLIST=()
DISPLAY_QUEUE=()
ALL_SONGS=$(FETCH_SONGS)

for num in "$@"; do
    if [[ "$num" =~ ^[0-9]+$ ]]; then
        line=$(echo "$ALL_SONGS" | sed -n "${num}p")
        if [ -n "$line" ]; then
            name=$(echo "$line" | cut -d'|' -f1)
            url=$(echo "$line" | cut -d'|' -f2)
            PLAYLIST+=("$url")
            DISPLAY_QUEUE+=("$name")
        fi
    fi
done

if [ ${#PLAYLIST[@]} -eq 0 ]; then
    echo "No valid songs selected."
    exit 1
fi

echo "Play Queue:"
echo "----------------------------------------"
for i in "${!DISPLAY_QUEUE[@]}"; do
    echo "  $((i+1)). ${DISPLAY_QUEUE[$i]}"
done
echo "----------------------------------------"
echo "Streaming via mpv..."

mpv --no-video "${PLAYLIST[@]}"



