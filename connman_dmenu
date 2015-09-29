#!/bin/bash
readonly SCAN_RESULT=/tmp/connman.scan
readonly STORAGE_PATH=/var/lib/connman
readonly VPN_STORAGE_PATH=/var/lib/connman-vpn

get_services() {
	if [[ -f $SCAN_RESULT ]]; then
		dmenu_notify 'another connman_dmenu is running'
		exit 1
	fi
	trap "rm -f $SCAN_RESULT" EXIT
	connmanctl enable wifi &>/dev/null
	connmanctl scan wifi &>/dev/null
	connmanctl services | \
		awk -F ' +' '{ service_id=$NF; $NF=""; $1=""; name=substr($0, 2, length-2); gsub(/[^a-zA-Z0-9-]/, "_", name) }
	name { print name, service_id }' > $SCAN_RESULT
}

# $1 = index
index_to_name() {
	[[ -f $SCAN_RESULT ]] || exit 1
	awk -v line="$1" 'NR == line { print $1 }' $SCAN_RESULT
}

# $1 = index
index_to_service() {
	[[ -f $SCAN_RESULT ]] || exit 1
	awk -v line="$1" 'NR == line { print $2 }' $SCAN_RESULT
}

# $1 = service id
get_service_security() {
	cut -d _ -f 5 <<<"$1"
}

# $1 = service id
get_service_signal() {
	connmanctl services "$1" | awk '$1 == "Strength" { print $3 }'
}

# $1 = service id
get_service_state() {
	connmanctl services "$1" | awk '$1 == "State" { print $3 }'
}

create_dmenu() {
	[[ -f $SCAN_RESULT ]] || exit 1
	echo 'setup vpn pptp'
	local order=1
	local name
	local service_id
	local security
	local signal
	local disconnect
	while read -r name service_id; do
		security=''
		signal=''
		disconnect=''
		[[ ! "$(get_service_state "$service_id")" =~ ^(idle|failure)$ ]] && disconnect='(disconnect)'
		case "$service_id" in
			wifi_*)
				security="$(get_service_security "$service_id")"
				signal="$(get_service_signal "$service_id")"
				;;
			vpn_*)
				security=vpn
				;;
		esac
		printf '%2s  %-40s%9s %-3s %s\n' "$order" "$name" "$security" "$signal" "$disconnect"
		(( order++ ))
	done < $SCAN_RESULT
}

# $1 = msg
dmenu_notify() {
	: | dmenu -p "$1 (press enter)"
}

# $1 = question
# $2 = var name
dmenu_ask() {
	IFS= read -r "$2" < <(: | dmenu -p "$1")
	if [[ ! "${!2}" ]]; then
		dmenu_notify "invalid $2"
		exit 1
	fi
}

if (( EUID != 0 )); then
	dmenu_notify 'please run it as root'
	exit 1
fi

get_services
index="$(create_dmenu | dmenu -l 10 -i -p 'select wifi service' | sed 's/^ *//g' | cut -d ' ' -f 1)"

if [[ "$index" == setup ]] ; then
	# create vpn mode
	dmenu_ask 'name this PPTP VPN' name
	name="${name// /_}"
	dmenu_ask 'please provide VPN domain' domain
	dmenu_ask 'please provide identity' identity
	dmenu_ask 'please provide password' password
	cat > "$VPN_STORAGE_PATH/$name.config" <<-EOF
	[provider_$name]
	Type = PPTP
	Name = $name
	Host = $(dig +short A "$domain" | sort -n | head -n1)
	Domain = $domain
	PPTP.User = $identity
	PPTP.Password = $password
	EOF
	dmenu_notify "VPN $name is created"
	exit 0
fi

service_id="$(index_to_service "$index")"
[[ "$service_id" ]] || exit 1

name="$(index_to_name "$index")"
echo "$name { $service_id }"

if [[ ! "$(get_service_state "$service_id")" =~ ^(idle|failure)$ ]]; then
	connmanctl disconnect "$service_id"
	dmenu_notify "$name disconnected"
	exit 0
fi

security="$(get_service_security "$service_id")"

# create service file for encryption
if [[ "$security" =~ ^(ieee8021x|psk|wep)$ ]]; then
	config_file="$STORAGE_PATH/$name-$security.config"
	if [[ -f "$config_file" && no != "$(echo -e 'yes\nno' | dmenu -p 'use previous profile?')" ]]; then
		echo "use old profile: $config_file"
	else
		dmenu_ask 'please provide password' password
		case "$security" in
			ieee8021x)
				dmenu_ask 'please provide identity' identity
				case "$(echo -e 'PEAP/MSCHAPV2\nTTLS/PAP' | dmenu -p 'please specify EAP type')" in
					PEAP/MSCHAPV2)
						eap=peap
						phase2=MSCHAPV2
						;;
					TTLS/PAP)
						eap=ttls
						phase2=PAP
						;;
					*)
						dmenu_notify 'invalid EAP'
						exit 1
						;;
				esac
				cat > "$config_file" <<-EOF
				[service_$service_id]
				Type = wifi
				Name = $name
				EAP = $eap
				Phase2 = $phase2
				Identity = $identity
				Passphrase = $password
				EOF
				;;
			psk|wep)
				cat > "$config_file" <<-EOF
				[service_$service_id]
				Type = wifi
				Name = $name
				Passphrase = $password
				EOF
				;;
		esac
		chmod 600 "$config_file"
	fi
fi

connman_msg="$(timeout 10 connmanctl connect "$service_id" 2>&1 | head -n 1)"
if [[ "$connman_msg" == Connected* ]]; then
	dmenu_notify "connected to $name"
else
	error_msg='automatic timeout for connman_dmenu'
	[[ "$connman_msg" ]] && error_msg="$(cut -d ' ' -f 3- <<<"$connman_msg")"
	dmenu_notify "cannot connect to $name ($error_msg)"
fi
