#!/bin/sh
# YTMP: a shell script that plays (and downloads) music from youtube with an extensive queue manager using fzf, vim, or cli
# DEPS: yt-dlp, mpv, socat, fzf, xargs, find, sed, bc, ed, coreutils, (n/vim), (pipe-viewer and awk for playlist search), (curl for thumbnails)

conf="/home/$USER/Music/ytmp/conf"
[ -e "$conf" ] && . "$conf"

# if to play one song after another by default
[ -z "$daemonize" ] && daemonize="1" # 1=true; 0/anything else=false

# maximum amount of times to play a song before downloading it
# set to a big number if not wanting this feature
[ -z "$max_stream_amount" ] && max_stream_amount=4
# the maximum a length a file can be (in seconds) if it's to be downloaded
[ -z "$max_len_for_dl" ] && max_len_for_dl='600'

[ -z "$download_thumbnails" ] && download_thumbnails="0" # 1=true; 0/anything else=false
[ -z "$notifications" ] && notifications="1" # 1=true; 0/anything else=false

[ -z "$prefix" ] && prefix="/home/$USER/Music/ytmp"

[ -z "$dl_dir" ] && dl_dir="$prefix/downloads"
[ -z "$songs_dir" ] && songs_dir="$dl_dir/songs"
[ -z "$thumb_dir" ] && thumb_dir="$dl_dir/thumbnails"
[ -z "$played_urls" ] && played_urls="$prefix/played_urls"
[ -z "$played_artists" ] && played_artists="$prefix/played_artists"
[ -z "$played_titles" ] && played_titles="$prefix/played_titles"
[ -z "$search_history" ] && search_history="$prefix/search_history"
[ -z "$queue" ] && queue="$prefix/queue"

# this is run on everytime a song is played so if you want certain
# mpv settings to remain consistent (volume, loop, seek, etc), then put them in this file
[ -z "$run_on_next" ] && run_on_next="/home/$USER/Music/ytmp/run_on_next"

mkdir -p "$prefix" "$dl_dir" "$songs_dir" "$thumb_dir"

# regex to match what's currently playing
playing_regex="^.\{11\} \*\*\*.*\*\*\*$\|^.\{12\} \*\*\*.*\*\*\*$"

# regex to match <id> ***
playing_marker_regex="^\(.\{11\}\) \*\*\*\|^\(.\{12\}\) \*\*\*"

helper () {
cat >&2 << CAT_EOF
Usage: ytmp [<search>]/s [# of results]/se [# of results]/S [<search>]/sp <search>/a <local path/dir/url>/ee # OR v/ls/m [p|l|#[+|-#]] [p|l|#[+|-#]]/E OR -l/-p OR n/p/pl/pf/mln/mfn/l [#|s]/P OR -r/-d

On first installing ytmp, there won't be any history to select from so either pass arguements from the cli with ytmp [S] <search> or to search what's in the input field press ctrl-/ or to background search press ctrl-a / ctrl-e (difference explained below)

The difference between s and se/S: se/S searches without the added jargon 'auto-generated provided to youtube' which usually fetches results from youtube music. If you just pass the query from the command line or just run ytmp without options, it will add the jargon to your search. ytmp and ytmp S also accept arguements as search if you don't want to enter fzf for search.

s view search results and select (can select multiple). can specify amount of search
   results to return with a following arguement; defaults to 5.

se search exactly. view search results and select (can select multiple). can specify amount of search
   results to return with a following arguement; defaults to 5.

S search exactly - no result selection.
sp search for playlists (requires https://github.com/trizen/pipe-viewer/); tab: see videos in playlist

fzf options for search (ytmp [S/se/s]):
			ctrl-r: replace input field with selection
			ctrl-x: send input field as search (none of the selections are passed)
			enter: search only the selections (query not included)
			tab: select
			shift-tab: deselect
			ctrl-c/esc: abort

		query+selections are only available as background search due to fzf limitations. when you press the binding there isn't any indication that the search has passed through so you can assume that fact and just abort fzf. also these options are unavailalbe or mean different things for se/s:
			ctrl-s: search query in the background
			alt-s: search query and selection in the background
			ctrl-z: search query in counterpart of current search in the background
			alt-z: search query and selection in counterpart of current search in the background

		for ytmp v/vv/se/s:
			ctrl-s: search query in background
			alt-s: search query in background with ytmp S

a add playlist, path, or directory (pass them as arguement - accepts many of all the kinds mentioned)

-p pause
-l loop what's currently playing

l [n|s] play the song that was played before this one or n before this one or pass s to select from the history file with fzf
w toggle mpv window. don't press q to close the window because that will close the file as well instead run ytmp w again to close it.
mfn queue last entry to play next
mln queue last entry to play next
e play entry #
pf play first entry
pl play last entry
n play next on queue
p play prev on queue
P fuzzy search string in queue and play match

v view queue; keybinds:
	   home: first; end: last
	   shift-up: move entry up one; shift-down: move entry down one; shift-right: play entry
	   shift-left: delete entry; page-up: move entry to after currently playing
	   page-down: move entry to before currently playing; enter: play entry and quit fzf
	   alt-m: ytmp m; ctrl-\: ytmp E; ctrl-6: ytmp mln; ctrl-]: ytmp w
	   alt-v: up one page; ctrl-v: down one page
	   ctrl-alt-p: up half page; ctrl-alt-n: down half page
	   left-click: play entry; right-click: move entry to after currently playing
	   alt-j: jump; ctrl-s: search query in background
	   alt-s: search query in background with ytmp S
	   ctrl-alt-d: download selection; alt-r: reload queue
	   ctrl-o: open song in web browser

vv view queue with details about the video (same keybinds as v option)

ls show a numbered view of the entries
m [p|l|#[+|-#]] [p|l|#[+|-#]] move entry. l means last, p means currently playing. examples: ytmp m 10 +2 or ytmp m p+3 l-1 or ytmp m l-1 p-2 or ytmp m l p+1

-shuf runs shuf on queue and overwrites it.

-d if ytmp does not run in the background automatically, run it by sending this arg
     so things continue playing one after another
-n get notified when mpv exits (song finishes)
-r play random entries; a range can be specified with two separate arguements
-dl download song # (or p|l[-+#] like 'm')

-rq delete queue file
-qa quit audio
-ky kill ytmp
-ka do all the above
-kd kill daemon

E edit the queue in nvim
vim key bindings:
__________________________________________________________
mapleader = '<Space>'
<leader>v source $XDG_CONFIG_HOME/nvim/init.vim
<leader>s source $XDG_CONFIG_HOME/nvim/ytmp.vim
<leader>c edit $XDG_CONFIG_HOME/nvim/ytmp.vim
<leader>n edit $USER/Music/ytmp/run_on_nextZ

<Up> move entry up 1
<Down> move entry down 1
<Left> move entry right before currently playing
<Right> move entry right next to currently playing
<S-Up> move last entry right before currently playing
<S-Down> move last entry right after currently playing
<S-Right> move entry to the end
<S-Left> move entry to the start
<Enter> play entry

d delete line
r reload file
R remove stars from currently playing entry
W write file
J go to currently playing

<C-t> :te
<C-y> :te ytmp
<C-w> :te ytmp S
<C-s> :te ytmp v
__________________________________________________________

h/help/-h/--help show this help

One can communicate with mpv through the ipc server the script opens at /tmp/mpvsocketytmp. See https://pastebin.com/23PXxpiD for examples.
CAT_EOF
exit
}

# gets the artists listed in the description and places them in $played_artists if first time seeing $id. artists are usually separated by '·' and ':'. also puts things in the queue file.
register () {
	id="$1"
	[ -n "$2" ] && title="$2" || title=$( yt-dlp -e "https://www.youtube.com/watch?v=$id" )
	echo "Adding '$title'"
	dt="$( date +'%m/%d/%y+%T' )"

	if ( ls $songs_dir/*"${id}"* >/dev/null 2>&1 ); then ( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue;
	elif ! grep -Fq "$id" "$played_urls"; then
		desc=$( yt-dlp --get-description "https://www.youtube.com/watch?v=$id" )
		echo "$title" >> $played_titles
		echo "$id 0 $dt $title" >> $played_urls
		if ( echo "$desc" | grep -q '·' ); then
			artist=$( echo "$desc" | grep '·' | sed -e 's/^[^·]*·//g' -e 's/·/\\\n/g' -e 's/\\//g' | xargs -0 printf '%b' )
			echo "$artist" | while read -r f; do
				if ! grep -q "$f" $played_artists; then
					echo "$f" >> $played_artists
				fi
			done
		fi
		if ( echo "$desc" | grep -q ':' ); then
			artist="$( echo "$desc" | sed -e  '/.*:/!d' -e 's/.*://p' | sed -e '/\./d' -e '/[0-9]/d' -e 's/\\//g' | sort -u )"
			echo "$artist" | while read -r f; do
				if ! grep -q "$f" $played_artists; then
					echo "$f" >> $played_artists
				fi
			done
		fi
		( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue;
	else ( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue; fi
}

# add urls, playlists, dirs, paths
add () {
	search="$( echo $* | cut -s -d' ' -f2- )"
	[ -z "$search" ] && echo "provide a url to a playlist or a path or a directory" && return

	for f in $search; do
		if ! echo "$f" | grep -q 'http.*\.com'; then
			if [ -d "$f" ]; then
				find "$f" | sed 1d | while read -r f; do
					echo "Adding '$f'"
					entry="$( cksum --untagged --algorithm=blake2b -l 48 "$f" | sed -E 's@\W @ @' )"
					grep -Fq "$entry" "$queue" || echo "$entry" >> "$queue"
				done
			elif [ -e "$f" ]; then
				echo "Adding '$f'"
				entry="$( cksum --untagged --algorithm=blake2b -l 48 "$f" | sed -E 's@\W @ @' )"
				grep -Fq "$entry" "$queue" || echo "$entry" >> "$queue"
			fi
		else
			ids="$( yt-dlp --get-id "$f" )"
			echo "$ids" | while read -r id; do
				title=$( yt-dlp -e "https://www.youtube.com/watch?v=$id" )
				echo "Adding '$title'"
				! grep -Fq "$id" "$played_urls" && dt="$( date +'%m/%d/%y+%T' )" && echo "$id 0 $dt $title" >> $played_urls
				if ( ls $songs_dir/*"${id}"* >/dev/null 2>&1 ); then ( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue;
				else ( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue; fi
			done
		fi
	done
}

# a separate function for 'S [<search>]' don't remember why...
search_exact () {
	songs_dir_ls=$( ls $songs_dir | cut -d' ' -f2- | sed 's/\.webm$//' )
	entries=$( cat $search_history $played_titles $played_artists )
	[ -n "$( echo $* | cut -s -d' ' -f3 )" ] && query=$( echo "$*" | cut -d' ' -f2- ) || query=$( echo "$entries $songs_dir_ls" | fzf --layout=reverse --height 40% -m --bind='ctrl-z:execute-silent(setsid -f ytmp {q}),alt-z:execute-silent(setsid -f ytmp {+} {q} ),alt-a:beginning-of-line,alt-e:end-of-line,alt-s:execute-silent(setsid -f ytmp S {+} {q} ),tab:select+clear-query,return:accept,ctrl-r:replace-query,ctrl-x:print-query,ctrl-s:execute-silent(setsid -f ytmp S {q})' )
	query=$( echo "$query" | tr "\n" ' ' | sed -e 's/[ \t]*$//' )
	[ -z "$query" ] && return
	echo "Searching for '$query'"
	( ! grep -Fq "$query" $queue ) && ( ls $songs_dir/*"${query}"* >/dev/null 2>&1 ) && find $songs_dir -name "*${query}*" -type f -print | xargs -d '\n' -n 1 basename | sed 's/\.webm$//' >> $queue || id=$( yt-dlp --get-id ytsearch1:"$query" )
	title=$( yt-dlp -e "https://www.youtube.com/watch?v=$id" )
	echo "Adding '$title'"
	( ! grep -Fq "$id" $queue) && echo "$id $title" >> $queue
	( ! grep -Fq "$query" $search_history ) && ( ! echo "$songs_dir_ls" | grep -Fq "$query" ) && echo "$query" >> $search_history
}

search_youtube () {
	songs_dir_ls=$( ls $songs_dir | cut -d' ' -f2- | sed 's/\.webm$//' )
	entries=$( cat $search_history $played_titles $played_artists )
	# if no arguements are passed but just words, search with the words
	[ "$1" = 'ss' ] && query=$( echo "$*" | cut -d' ' -f2- ) || query=$( echo "$entries $songs_dir_ls" | fzf --layout=reverse --height 40% -m --bind='ctrl-z:execute-silent(setsid -f ytmp S {q}),alt-z:execute-silent(setsid -f ytmp S {+} {q} ),alt-a:beginning-of-line,alt-e:end-of-line,alt-s:execute-silent(setsid -f ytmp {+} {q}),tab:select+clear-query,return:accept,ctrl-r:replace-query,ctrl-x:print-query,ctrl-s:execute-silent(setsid -f ytmp {q})' )
	query=$( echo "$query" | tr "\n" ' ' | sed -e 's/[ \t]*$//' )
	[ -n "$query" ] && echo "Searching for '$query'"
	( ! grep -Fxq "$query" $search_history ) && ( ! echo "$songs_dir_ls" | grep -Fq "$query" ) && echo "$query" >> $search_history

	# allows user to browse search results and select
	if [ "$1" = 'se' ]; then
		[ -n "$2" ] && ids_titles=$( yt-dlp --get-id -e ytsearch${2}:"$query" ) || ids_titles=$( yt-dlp --get-id -e ytsearch5:"$query" )
		# separates ids and titles. shows only $titles to user and then uses the number of the line $title is found on
		# to get the correct id.
		ids=$( echo "$ids_titles" | sed -n 'n;p' )
		# i don't know how to do it in a more 'proper' way so to get the ids for preview, i put the ids in a file and then use the fzf index (+1) to get the proper id for the description
		echo "$ids" > /tmp/ytmpids
		titles=$( echo "$ids_titles" | sed -n 'p;n' | grep -n '' )
		# sel=$( printf "$titles" | fzf -m --preview="echo {n}+1 | bc | xargs -I ',,' sed -n ,,p /tmp/ytmpids | xargs -I ',,' pipe-viewer --extract 'NOM\t*TITLE*\nCNL\t*AUTHOR*\nPUB\t*PUBLISHED*\nDUR\t*TIME*\nURL\t*URL*\n\n\tDESCRIPTION\n*DESCRIPTION*' -id ,," --height 60% --preview-window='right,70%' )
		sel=$( printf '%b' "$titles" | fzf --layout=reverse --height 40% -m --preview="echo {n}+1 | bc | xargs -I ',,' sed -n ,,p /tmp/ytmpids | xargs -I ',,' yt-dlp --print title --print duration_string --print description ,," --height 60% --preview-window='right,70%' )

		echo "$sel" | while read -r one; do
			num=$( printf '%b' "$titles" | grep -Fx "$one" | cut -d':' -f1 )
			id=$( printf '%b' "$ids" | sed -n "${num}"p )
			echo "Adding '$( echo $one | cut -d':' -f2- )'"
			( ! grep -Fq "$id" $queue) && echo "$id $( echo $one | cut -d':' -f2- )" >> $queue
		done
	elif [ "$1" = 's' ]; then
		[ -n "$2" ] && ids_titles=$( yt-dlp --get-id -e ytsearch${2}:"$query auto-generated provided to youtube" ) || ids_titles=$( yt-dlp --get-id -e ytsearch5:"$query auto-generated provided to youtube" )
		# separates ids and titles. shows only $titles to user and then uses the number of the line $title is found on
		# to get the correct id.
		ids=$( echo "$ids_titles" | sed -n 'n;p' )
		titles=$( echo "$ids_titles" | sed -n 'p;n' | grep -n '' )
		# i don't know how to do it in a more 'proper' way so to get the ids for preview, i put the ids in a file and then use the fzf index (+1) to get the proper id for the description
		echo "$ids" > /tmp/ytmpids
		# sel=$( printf "$titles" | fzf -m --preview="echo {n}+1 | bc | xargs -I ',,' sed -n ,,p /tmp/ytmpids | xargs -I ',,' pipe-viewer --extract 'NOM\t*TITLE*\nCNL\t*AUTHOR*\nPUB\t*PUBLISHED*\nDUR\t*TIME*\nURL\t*URL*\n\n\tDESCRIPTION\n*DESCRIPTION*' -id ,," --height 60% --preview-window='right,70%' )
		sel=$( printf '%b' "$titles" | fzf --layout=reverse --height 40% -m --preview="echo {n}+1 | bc | xargs -I ',,' sed -n ,,p /tmp/ytmpids | xargs -I ',,' yt-dlp --print title --print duration_string --print description ,," --height 60% --preview-window='right,70%' )

		echo "$sel" | while read -r one; do
			num=$( printf '%b' "$titles" | grep -Fx "$one" | cut -d':' -f1 )
			id=$( printf '%b' "$ids" | sed -n "${num}"p )
			register "$id" "$( echo $one | cut -d':' -f2- )"
		done
	elif [ -n "$query" ]; then
		( ! grep -Fq "$query" $queue ) && ( ls $songs_dir/*"${query}"* >/dev/null 2>&1 ) && find $songs_dir -name "*${query}*" -type f -print | xargs -d '\n' -n 1 basename | sed 's/\.webm$//' >> $queue || id=$( yt-dlp --get-id ytsearch1:"$query auto-generated provided to youtube" )
		[ -n "$id" ] && register "$id"
	fi
}

search_playlist () {
	[ -z "$2" ] && echo 'provide a search string as arguement' || ( query=$( echo "$*" | cut -s -d' ' -f2- ) && pipe-viewer --no-interactive --results=50 -sp --custom-playlist-layout='*VIDEOS*VIDS *TITLE* *URL*' "$query" | fzf --layout=reverse --height 40% --bind='alt-a:beginning-of-line,alt-e:end-of-line,tab:execute(echo {} | awk "{print $NF}" | xargs -0 -I ",," pipe-viewer --results=50 --custom-layout="*AUTHOR* *TIME* *TITLE*" --no-interactive ",," | fzf)' | awk '{print $NF}' | xargs -r -0 setsid -f ytmp a >/dev/null 2>&1 )
}

# ticks up the amount a song has been played by one. $played_urls has the format: id amount_played date_played 'dl'(if downloaded) title
# also downloads the file once it exceeds $max_stream_amount and it can't be found in the $songs_dir
upd_amnt_played () {
	[ -n "$1" ] && id="$1" || id=$( grep "$playing_regex" $queue | cut -d' ' -f1 )
	[ -n "$1" ] && title=$( echo "$*" | cut -d' ' -f2- ) || title=$( sed -n -e "/$playing_regex/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )
	dt="$( date +'%m/%d/%y+%T' )"
	amnt_played="$( grep -F "$id" "$played_urls" | head -1 | cut -d' ' -f2 )"
	amnt_played=$((amnt_played + 1))
	sed -i "/$id/d" $played_urls
	[ "$( printf "$id" | wc -m )" = '12' ] && echo "$id $amnt_played $dt $title" >> $played_urls && return
	[ -z "$id" ] && return
	if [ "$amnt_played" -gt "$max_stream_amount" ]; then
		if ( ls $songs_dir/*"${id}"* >/dev/null 2>&1 ); then
			echo "$id $amnt_played $dt dl $title" >> $played_urls
		else
			len=$( yt-dlp --get-duration "https://www.youtube.com/watch?v=$id" | sed -E 's/(.*):(.+):(.+)/\1*3600+\2*60+\3/;s/(.+):(.+)/\1*60+\2/' | bc )
			[ $len -lt $max_len_for_dl ] && ( setsid -f yt-dlp -f bestaudio -o "$songs_dir/%(id)s %(title)s.%(ext)s" "https://www.youtube.com/watch?v=$id" >/dev/null 2>&1 && echo "Downloading '$title'. You have listened to it $amnt_played times." && echo "$id $amnt_played $dt dl $title" >> $played_urls ) || echo "$id $amnt_played $dt $title" >> $played_urls
		fi
	else
		ls $songs_dir/*"${id}"* >/dev/null 2>&1 && echo "$id $amnt_played $dt dl $title" >> $played_urls || echo "$id $amnt_played $dt $title" >> $played_urls

	fi
}

dl_thumbnail () {
	if [ "$download_thumbnails" = '1' ]; then
		id=$( grep "$playing_regex" $queue | cut -d' ' -f1 )
		[ "$( printf "$id" | wc -m )" != '11' ] && return
		if ( ! ls $thumb_dir/*"${id}"* >/dev/null 2>&1 ); then
			title=$( sed -n -e "/$playing_regex/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )
			yt-dlp --get-thumbnail "https://www.youtube.com/watch?v=$id" | xargs curl --output "$thumb_dir/$id $title.webp"
			ln -fs $thumb_dir/*"${id}"* /tmp/muscover.webp
		else
			ln -fs $thumb_dir/*"${id}"* /tmp/muscover.webp
		fi
	fi
}

# plays the $id passed to it. if no arguements, it sets id to whatever is surrounded by ***
play () {
	[ -n "$1" ] && id="$1" || id=$( grep "$playing_regex" $queue | cut -d' ' -f1 )
	[ -z "$id" ] && return
	# if the length of id (first field) is 12 then it is a checksum of a local file and not a youtube id so play it and exit
	if [ "$( printf "$id" | wc -m )" = '12' ]; then
		setsid -f mpv --x11-name='ytmp_mpv' --no-terminal --vid=no --input-ipc-server=/tmp/mpvsocketytmp "$( sed -n -e "/$id/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )"
		upd_amnt_played
		return
	fi
	if ( ls $songs_dir/*"${id}"* >/dev/null 2>&1 ); then
		find $songs_dir/*"${id}"* -print0 | xargs -0 -I ',,' setsid -f mpv --x11-name='ytmp_mpv' --no-terminal --vid=no --input-ipc-server=/tmp/mpvsocketytmp ",,"
		dl_thumbnail
	else
		setsid -f mpv --x11-name='ytmp_mpv' --no-terminal --vid=no --input-ipc-server=/tmp/mpvsocketytmp --ytdl-format='bestaudio' "https://www.youtube.com/watch?v=$id"
		dl_thumbnail
	fi
	sh "$run_on_next" >/dev/null 2>&1
	echo "Playing '"$( sed -n -e "/$playing_regex/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )"'"
	[ "$notifications" = '1' ] && notify-send 'Now Playing' "$( sed -n -e "/$playing_regex/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )"
	upd_amnt_played
}

playlist () {
	# plays next or previous song to currently playing
	if [ "$1" = 'c' ]; then
		index=$( grep -n "$playing_regex" "$queue" | head -1 | cut -d':' -f1 )
		sed -i -e "${index}s/$playing_marker_regex/\1\2 /" -e 's/\*\*\*$//' "$queue"
		[ "$2" = "n" ] && index=$(( index + 1 ))
		[ "$2" = "p" ] && index=$(( index - 1 ))
		lncount=$( grep -c '.' "$queue" )
		[ "$index" = 0 ] && index="$lncount"
		[ "$index" -gt "$lncount" ] && index=1
		title=$( sed -n ${index}p $queue | cut -d' ' -f2- | sed -e 's`[][\\/.*^$&]`\\&`g' )
		sed -i "${index}s/$title/\*\*\*$title\*\*\*/" "$queue"
		echo quit | socat - /tmp/mpvsocketytmp
		play
	# puts stars around exact title and removes stars from any other lines
	elif [ "$1" = 'ee' ]; then
		title="$( echo "$2" | sed -e 's`[][\\/.*^$&]`\\&`g' )"
		sed -i -e "/$playing_regex/s/$playing_marker_regex/\1\2 /" -e 's/\*\*\*$//' "$queue"
		sed -i "s/\(^.\{11\}\) $title$\|\(^.\{12\}\) ${title}$/\1\2 \*\*\*${title}\*\*\*/" $queue
		echo quit | socat - /tmp/mpvsocketytmp
		play
	# puts stars around line # and removes stars from any other lines
	elif [ "$1" = 'e' ]; then
		[ "$3" = 'f' ] && line=$(( $4 + 1 )) || line="$3"
		sed -i -e "/$playing_regex/s/$playing_marker_regex/\1\2 /" -e 's/\*\*\*$//' "$queue"
		# title="$( sed -n -e ${line}'s/\(^.\{11\}\) \(.*\)\|\(^.\{12\}\) \(.*\)/\2\4/' -e 's`[][\\/.*^$&]`\\&`gp' "$queue" )"
		title="$( sed -n "${line}p" "$queue" | cut -d' ' -f2- | sed -e 's`[][\\/.*^$&]`\\&`g' )"
		sed -i "${line}s/${title}$/\*\*\*${title}\*\*\*/" $queue
		echo quit | socat - /tmp/mpvsocketytmp
		play

	# checks if queue is not empty but nothing is playing, it will set stars around the first line and call play
	# it also checks if mpv is closed (i.e. not busy and finished playing last song)
	# then it calls ytmp n (playlist c n) to play the next entry
	elif [ -z "$1" ]; then
		while true; do
			if [ "$( grep -c '.' $queue )" -gt 0 ]; then
				if ! grep -q "$playing_regex" $queue; then
					id="$( head -1 "$queue" | cut -d' ' -f1 )"
					title="$( head -1 "$queue" | cut -d' ' -f2- | sed -e 's`[][\\/.*^$&]`\\&`g' )"
					sed -i "s/$title/\*\*\*$title\*\*\*/" "$queue"
					play "$id"
				else
					sleep 1.5 && ( ! pgrep -fa 'mpv.*--input-ipc-server=/tmp/mpvsocketytmp' >/dev/null 2>&1 ) && playlist c n
				fi
			fi
			sleep 1
		done
	fi
}

# all the things that can be done to the arrangement of the queue. managed from fzf.
queue_manager () {
	# presents an fzf list of the current queue.
	if [ "$1" = 'v' ]; then
		cmd="cat $queue | cut -d' ' -f2-"
		cat $queue | cut -d' ' -f2- | fzf --layout=reverse --height 40% --cycle --bind="ctrl-]:execute-silent(ytmp w),ctrl-r:replace-query,home:first,end:last,alt-r:reload(cat '$queue'),ctrl-alt-d:execute-silent(ytmp -dl {n} f),alt-s:execute-silent(setsid -f ytmp S {q}),ctrl-s:execute-silent(setsid -f ytmp {q}),ctrl-j:down,alt-j:jump,shift-left:execute-silent(ytmp dd {n})+reload($cmd),shift-up:execute-silent(ytmp up {n})+reload($cmd)+up,shift-down:execute-silent(ytmp dn {n})+reload($cmd)+down,return:execute-silent(ytmp e f {n})+abort,shift-right:execute-silent(ytmp e f {n})+reload($cmd),pgdn:execute-silent(ytmp nn {n})+reload($cmd),pgup:execute-silent(ytmp pp {n})+reload($cmd),ctrl-v:page-down,alt-v:page-up,ctrl-alt-p:half-page-up,ctrl-alt-n:half-page-down,left-click:execute-silent(ytmp e f {n})+abort,right-click:execute-silent(ytmp nn {n})+reload($cmd),ctrl-^:execute-silent(ytmp mln)+reload($cmd),ctrl-\:execute(ytmp E)+reload($cmd),alt-m:execute(ytmp m f {n})+reload($cmd),ctrl-o:execute-silent(ytmp opinbrwsr {n})"

	elif [ "$1" = 'vv' ]; then
		cat "$queue" | fzf --layout=reverse --height 40% --cycle --bind="ctrl-]:execute-silent(ytmp w),home:first,end:last,alt-r:reload(cat '$queue'),ctrl-alt-d:execute-silent(ytmp -dl {n} f),ctrl-r:replace-query,alt-s:execute-silent(setsid -f ytmp S {q}),ctrl-s:execute-silent(setsid -f ytmp {q}),ctrl-j:down,alt-j:jump,shift-left:execute-silent(ytmp dd {n})+reload(cat '$queue'),shift-up:execute-silent(ytmp up {n})+reload(cat '$queue')+up,shift-down:execute-silent(ytmp dn {n})+reload(cat '$queue')+down,return:execute-silent(ytmp e f {n})+abort,shift-right:execute-silent(ytmp e f {n})+reload(cat '$queue'),pgdn:execute-silent(ytmp nn {n})+reload(cat '$queue'),pgup:execute-silent(ytmp pp {n})+reload(cat '$queue'),ctrl-v:page-down,alt-v:page-up,ctrl-alt-p:half-page-up,ctrl-alt-n:half-page-down,left-click:execute-silent(ytmp e f {n})+abort,right-click:execute-silent(ytmp nn {n})+reload(cat '$queue'),ctrl-^:execute-silent(ytmp mln)+reload(cat '$queue'),ctrl-\:execute(ytmp E)+reload(cat '$queue'),alt-m:execute(ytmp m f {n})+reload(cat '$queue'),ctrl-o:execute-silent(ytmp opinbrwsr {n})" --height 100% --preview='( printf "#"; echo "{n}+1" | bc ) | tr -d "\n"; printf "\n"; echo {} | cut -s -d" " -f1 | xargs -I "{}" yt-dlp --print title --print duration_string --print description "{}"' --preview-window='right,50%'

	elif [ "$1" = 'm' ]; then
		if [ "$2" = 'f' ]; then
			read place
			sel="$( echo "1+${3}" | bc )"
		else
			place="$3"
			sel="$2"
		fi

		[ -z "$place" ] && return
		sel="$( echo "$sel" | sed s/p/"$( grep -n "$playing_regex" "$queue" | head -1 | cut -d':' -f1 )"/ | sed s/l/"$( grep -c '.' "$queue" )"/ | bc )"
		place="$( echo "$place" | sed s/p/"$( grep -n "$playing_regex" "$queue" | head -1 | cut -d':' -f1 )"/ | sed s/l/"$( grep -c '.' "$queue" )"/ | bc )"

		ed -s "$queue" >/dev/null <<ED_END
${sel}m${place}
wq
ED_END

	# move entry to right after whats currently being played (in ***)
	elif [ "$1" = 'nn' ]; then
		ed -s "$queue" >/dev/null <<ED_END
/$playing_regex/
${3}+m.
wq
ED_END
	# move entry to right before whats currently being played (in ***)
	elif [ "$1" = 'pp' ]; then
		ed -s "$queue" >/dev/null <<ED_END
/$playing_regex/
${3}+m.-
wq
ED_END
	# move entry up one
	elif [ "$1" = 'up' ]; then
		ed -s "$queue" >/dev/null <<ED_END
${3}+m${3}-
wq
ED_END
	# move entry down one
	elif [ "$1" = 'dn' ]; then
		ed -s "$queue" >/dev/null <<ED_END
${3}+m${3}+2
wq
ED_END
	# delete entry
	elif [ "$1" = 'dd' ]; then
		ed -s "$queue" >/dev/null <<ED_END
${3}+d
wq
ED_END
	fi
}

fuzzy_play () {
	query=$( echo "$*" | cut -d' ' -f2 )
	match=$( cat "$queue" | fzf -f "$query" | head -1 | cut -d' ' -f2- )
	[ -n "$match" ] && playlist ee "$match" || echo 'match not found'
}

play_history () {
	[ -z "$1" ] && tail -2 "$played_urls" | head -1 | cut -d' ' -f1 | xargs -I '{}' ytmp P '{}' && return
	if ( echo "$1" | grep -qxE '[0-9]*' ); then
		tail "-$1" "$played_urls" | head -1 | cut -d' ' -f1 | xargs -I '{}' ytmp P '{}'
	elif [ "$1" = 's' ]; then
		cat "$played_urls" | fzf --layout=reverse --height 40% --tac | cut -d' ' -f1 | xargs -I '{}' ytmp P '{}'
	else
		tail -2 "$played_urls" | head -1 | cut -d' ' -f1 | xargs -I '{}' ytmp P '{}'
	fi
}

rand_daemon () {
	while $( while true; do ( ! echo '{ "command": ["get_property", "pid"] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 ) && sleep .5 && break; done ); do
		if ( ! pgrep -fa 'mpv.*--input-ipc-server=/tmp/mpvsocketytmp' ); then
			if [ -n "$3" ]; then
				seq "$2" "$3" | shuf -n 1 | xargs ytmp e
			else grep -c '' "$queue" | xargs seq | shuf -n 1 | xargs ytmp e;
			fi
		fi
	sleep 3
	done
}

download_song () {
	sel="$( echo $2 | sed s/p/"$( grep -n "$playing_regex" "$queue" | head -1 | cut -d':' -f1 )"/ | sed s/l/"$( grep -c '.' "$queue" )"/ | bc )"
	[ "$3" = 'f' ] && sel=$(( sel + 1 ))
	# [ "$( read -eq Are you sure you want to download '$( sed ${sel}p )'? (y/n)" = 'n' ] && return
	id="$( sed -n ${sel}p "$queue" | cut -d' ' -f1 )"
	[ "$( printf "$id" | wc -m )" != '11' ] && return
	# len=$( yt-dlp --get-duration "https://www.youtube.com/watch?v=$id" | sed -E 's/(.*):(.+):(.+)/\1*3600+\2*60+\3/;s/(.+):(.+)/\1*60+\2/' | bc )
	title="$( sed -n -e "/$playing_regex/s/$playing_marker_regex//" -e 's/\*\*\*$//p' "$queue" )"
	( ! ls $songs_dir/*"${id}"* >/dev/null 2>&1 ) && setsid -f yt-dlp -f bestaudio -o "$songs_dir/%(id)s %(title)s.%(ext)s" "https://www.youtube.com/watch?v=$id" >/dev/null 2>&1 && echo "Downloading '$title'" || echo "There is already a file of that name downloaded."
}

open_in_browser () {
	selt=$( echo "$*" | cut -d' ' -f2 )
	index=$(( selt+1 ))
	id=$( sed -n "${index}p" "$queue" | cut -d' ' -f1 )
	setsid -f $BROWSER "https://www.youtube.com/watch?v=$id" >/dev/null 2>&1
}

toggle_mpv_window () {
	state=$( echo '{ "command": ["get_property", "force-window" ] }' | socat - /tmp/mpvsocketytmp | jq .data )
	[ "$state" = "false" ] && echo '{ "command": ["set_property", "force-window", true ] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 && echo '{ "command": ["keypress", "I"] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 && sleep .5 && echo '{ "command": ["keypress", "O"] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 || echo '{ "command": ["set_property", "force-window", false ] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1
}

case "${1}" in
	E) nvim -u "$XDG_CONFIG_HOME/nvim/ytmp.vim" --noplugin +/'\*\*\*.*\*\*\*' "$queue" ;;
	# the 'a' doesn't mean anything. it's there so the # is in the proper field.
	mfn) queue_manager nn a 0 ;;
	mln) queue_manager nn a '$-' ;;
	w) toggle_mpv_window ;;
	# the 'a' doesn't mean anything. it's there so the # is in the proper field.
	pf) playlist e a 1 ;;
	pl) playlist e a "$(  grep -c '.' "$queue" )" ;;
	-p) echo cycle pause | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 ;;
	v) queue_manager v ;;
	vv) queue_manager vv ;;
	s) search_youtube s "$2" ;;
	se) search_youtube se "$2" ;;
	S) search_exact "$*" ;;
	sp) search_playlist $* ;;
	a) add $* ;;
	n) playlist c n ;;
	p) playlist c p ;;
	e) playlist e $* ;;
	P) fuzzy_play "$*" ;;
	l) play_history $2 ;;
	dd) queue_manager dd $* ;;
	up) queue_manager up $* ;;
	dn) queue_manager dn $* ;;
	nn) queue_manager nn $* ;;
	pp) queue_manager pp $* ;;
	m) queue_manager m "$2" "$3" "$4" ;;
	fct) queue_manager fct $* ;;
	ls) grep --color=auto -n '' "$queue" ;;
	-shuf) shuf "$queue" -o "$queue" ;;
	opinbrwsr) open_in_browser "$*" ;;
	-dl) download_song $* ;;
	-d) playlist ;;
	-r) rand_daemon $* ;;
	-n) while true; do ( ! echo '{ "command": ["get_property", "pid"] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 ) && sleep .5 && break; done ;;
	-l) echo '{ "command": ["set_property", "loop", true] }' | socat - /tmp/mpvsocketytmp >/dev/null 2>&1 ;;
	h|help|-h|--help) helper ;;
	-rq) rm $queue ;;
	-qa) echo quit | socat - /tmp/mpvsocketytmp ;;
	-ky) pkill ytmp ;;
	-ka) echo quit | socat - /tmp/mpvsocketytmp; rm $queue; pkill ytmp ;;
	-kd) pkill -f 'ytmp -d' && exit ;;
	-kr) pkill -f 'ytmp -r' && exit ;;
	*) search_youtube ss "$*" ;;
esac

[ -z "$1" ] && search_youtube

[ "$daemonize" = 1 ] && ! pgrep -fa 'ytmp -d' && setsid -f ytmp -d >/dev/null 2>&1
