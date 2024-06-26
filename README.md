#!/bin/bash

# This script is used to check if given package can be auto-updated and optionally enable if it can be.
# NOTE: You should not trust this script if package uses commit hashes for versioning(eg. 'tsu') or
# TERMUX_PKG_VERSION is defined by us and not upstream(i.e we use some arbitrary version. For eg. when
# getting source files based on commit hash).

# The MIT License (MIT)

# Copyright (c) 2022 Aditya Alok <alok@termux.org>

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

set -e

TERMUX_SCRIPTDIR="$(realpath "$(dirname "$(readlink -f "$0")")/../..")"

_RESET_COLOR="\033[0m"
_BLUE="\033[1;34m"
_GREEN="\033[0;32m"
_RED="\033[0;31m"
_YELLOW="\033[0;33m"
_TEAL="\033[0;36m"

usage() {
	echo
	echo -e "${_BLUE}Usage:${_RESET_COLOR} $(basename "$0")" \
		"${_GREEN}[-h|--help]" \
		"[-s|--silent]" \
		"[--enable]" \
		"PACKAGE_DIR"
	echo
	echo -e "${_TEAL}Check if packages can be auto-updated and optionally enable if so."
	echo
	echo -e "${_RED}NOTE: ${_YELLOW}You should not trust this script if\n" \
		"\t${_BLUE}- ${_YELLOW}package uses commit hashes for versioning (eg. 'tsu') or\n" \
		"\t${_BLUE}- ${_GREEN}TERMUX_PKG_VERSION ${_YELLOW}is defined by us and not upstream, i.e we use some arbitrary version.\n" \
		"\t  For eg. when only getting source files based on commit hash."
	echo
	echo -e "${_BLUE}Options:"
	echo -e "${_GREEN}-h --help${_TEAL} Show this help message."
	echo -e "${_GREEN}--enable${_TEAL} Enable auto-update for package if check was successful." \
		"(writes ${_YELLOW}TERMUX_PKG_AUTO_UPDATE=true${_TEAL} to build.sh)"
	echo -e "${_GREEN}-s --silent${_TEAL} Do not print anything to stdout."
	echo
	echo -e "${_BLUE}Example:${_RESET_COLOR} $(basename "$0") x11-packages/xorg-server"
	echo
}

color_print() {
	[[ "${SILENT}" == "true" ]] && return 0
	echo -e "$1${_RESET_COLOR}"
}

warn() {
	color_print "${_BLUE}[${_YELLOW}*${_BLUE}]${_YELLOW} $*" >&2
}

error() {
	local exit="return"
	[[ "${1}" == "--exit" ]] && exit="exit" && shift
	color_print "${_BLUE}[${_RED}!${_BLUE}]${_RED} $*" >&2
	${exit} 1
}

info() {
	color_print "${_BLUE}[${_GREEN}+${_BLUE}]${_TEAL} $*"
}

_check_stderr() {
	local stderr="$1"

	local http_code
	http_code="$(grep "HTTP code:" <<<"${stderr}" | cut -d ':' -f2 | tr -d ' ')"

	if [[ -n "${http_code}" ]]; then
		[[ ${http_code} == 000 ]] && error --exit "Could not connect to server. Please check your connection."
		warn "Failed to get tag. [HTTP code: $http_code]"
		return "$http_code"
	elif grep -qE "ERROR: No '(latest-release-tag|newest-tag)'" <<<"${stderr}"; then
		return 2
	elif grep -q "ERROR: GITHUB_TOKEN" <<<"$stderr"; then
		error --exit "GITHUB_TOKEN is not set." # exit script on this error.
	else
		warn "$stderr"
		return 1
	fi
}

_check() {
	local from_where="$1" # Which api service to use?
	shift
	local stderr
	stderr="$(
		set -euo pipefail
		# shellcheck source=/dev/null
		. "${TERMUX_SCRIPTDIR}/scripts/updates/api/termux_${from_where}_api_get_tag.sh"
		. "${TERMUX_SCRIPTDIR}/scripts/updates/utils/termux_error_exit.sh"
		termux_"${from_where}"_api_get_tag "$@" 2>&1 >/dev/null
	)" || _check_stderr "${stderr}"
}

check() {
	local from_where="$1" # Can be either 'github' or 'gitlab'.
	local url="$2"
	local return_code=0

	if [[ ! ${src_url#git+} =~ ^https?://${from_where}.com ]]; then
		warn "Not a ${from_where} url: ${src_url#git+}"
		return 1
	fi

	_check "$from_where" "$url" || return_code="$?"

	if [[ "${url:0:4}" != "git+" ]] && [[ "$return_code" == "2" ]]; then
		warn "No 'latest-release-tag' found. Checking for 'newest-tag'..."
		return_code=3 # Set return code to 3 to indicate that TERMUX_PKG_UPDATE_TAG_TYPE='newest-tag'
		# should also be written to build.sh if we find 'newest-tag' in next line.
		_check "$from_where" "$url" "newest-tag" || return_code="$?"
	fi

	return "$return_code"
}

declare -a UNIQUE_PACKAGES
# NOTE: This function ignores Termux own packages like 'termux-api','termux-exec', etc.
# It should be manually checked.
repology() {
	local pkg_name="$1"
	if [[ -z "${UNIQUE_PACKAGES[*]}" ]]; then
		# NOTE: mapfile requires bash 4+
		mapfile -t UNIQUE_PACKAGES < <(
			curl -A "Termux update checker 1.0 (github.com/termux/termux-packages)" \
				--silent --location --retry 5 \
				--retry-delay 5 --retry-max-time 60 \
				"https://repology.org/api/v1/projects/?inrepo=termux&&repos=1" |
				jq -r 'keys[]'
		)

		[[ -z "${UNIQUE_PACKAGES[*]}" ]] && error --exit "Failed to get unique packages from repology.org"
	fi
	# shellcheck disable=SC2076
	if [[ ! " ${UNIQUE_PACKAGES[*]} " =~ " ${pkg_name} " ]]; then
		return 0 # Package is not unique, can be updated.
	else
		warn "Package '$pkg_name' is unique to Termux, cannot be auto-updated."
		return 1 # Package is unique, cannot be updated.
	fi
}

write_to_first_empty_line() {
	local -r file="$1"
	local -r line="$2"
	# Find first empty line from top of file:
	local -r first_empty_line=$(sed -n '/^$/=' "$file" | head -n 1)
	if [[ -n "$first_empty_line" ]]; then
		sed -i "${first_empty_line}i$line" "$file"
	else
		echo "$line" >>"$file"
	fi
}

test_pkg() {
	local pkg_dir="$1"
	local pkg_name
	pkg_name="$(basename "${pkg_dir}")"

	if [[ ! -f "${pkg_dir}/build.sh" ]]; then
		error --exit "Package '$pkg_name' does not exist."
	fi

	local vars
	vars="$(
		set +eu
		# shellcheck source=/dev/null
		. "$pkg_dir/build.sh" >/dev/null
		echo "src_url=\"${TERMUX_PKG_SRCURL}\";"
		echo "auto_update=\"${TERMUX_PKG_AUTO_UPDATE}\";"
	)"
	local src_url=""
	local auto_update=""
	eval "$vars"

	color_print "${_BLUE}==>${_YELLOW} $pkg_name"

	# Check if auto_update is not empty, then package has been already checked.
	if [[ -n "${auto_update}" ]]; then
		warn "Auto-update is already set to ${_BLUE}'${auto_update}'${_YELLOW} in build.sh."
		return 0
	elif [[ -z "${src_url}" ]]; then
		error "Could not find TERMUX_PKG_SRCURL."
	fi

	local checks=(
		"github"
		"gitlab"
		"repology"
	)
	local can_be_updated=false
	local tag_type=""
	local update_method=""
	for check in "${checks[@]}"; do
		info "Checking if package can be updated from $check..."
		local return_code=0

		if [[ "$check" != "repology" ]]; then
			check "$check" "$src_url" || return_code="$?"
		else
			# sleep to ensure we do not flood repology with requests
			sleep 1
			repology "$pkg_name" || return_code="$?"
			update_method="repology"
		fi

		if [[ "$return_code" == "0" ]]; then
			can_be_updated=true
			break
		elif [[ "$return_code" == "3" ]]; then
			can_be_updated=true
			tag_type="newest-tag"
			break
		fi
	done

	if [[ "${can_be_updated}" == "true" ]]; then
		info "Package can be auto-updated."
		if [[ "${ENABLE}" == "--enable" ]]; then
			info "Enabling auto-update..."
			write_to_first_empty_line "$pkg_dir/build.sh" "TERMUX_PKG_AUTO_UPDATE=true"
			if [[ -n "${tag_type}" ]]; then
				write_to_first_empty_line "$pkg_dir/build.sh" "TERMUX_PKG_UPDATE_TAG_TYPE=\"${tag_type}\""
			fi
			if [[ -n "${update_method}" ]]; then
				write_to_first_empty_line "$pkg_dir/build.sh" "TERMUX_PKG_UPDATE_METHOD=${update_method}"
			fi
			info "Done."
		fi
	else
		error "Package cannot be auto-updated."
	fi
}

if [[ $# -lt 1 ]] || [[ $# -gt 3 ]]; then
	error --exit "Invalid number of arguments. See --help for usage."
fi

while [[ $# -gt 0 ]]; do
	case "$1" in
	--enable)
		ENABLE="$1"
		;;
	-s | --silent)
		SILENT=true
		;;
	-h | --help)
		usage
		;;
	*)
		test_pkg "$1"
		[[ "${SILENT}" == "true" ]] || echo
		;;
	esac
	shift
done
<!---
Akhilakhilakku/Akhilakhilakku is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
