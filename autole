#! /bin/bash
#
# LET'S ENCRYPT AUTOMATION
#
# by Damia Soler
# Contact: damia (at) damia (dot) net
# https://blog.damia.net
#
# You will need bc, xargs, curl, apache2ctl, mail, openssl.
# Be sure to configure "LEEMAIL" below.

PATH=/sbin:/bin:/usr/sbin:/usr/bin

# Remaining days to expire before renewal
DAYSTORENEW=10

# Remaining days to expire before alert (this is supposed to be less than DAYSTORENEW)
DAYSTOALERT=5

# Email to alert if unable to renew (leave blank to suppress notification mails)
ALERTEMAIL=root@localhost

# Email account on Let's Encrypt
LEEMAIL=admin@example.com

WEBROOT=/var/www/html/letsencrypt
ACMEPATH=.well-known/acme-challenge
LEBIN=/root/.local/share/letsencrypt/bin/letsencrypt
APACHE_RELOAD=false


function checkAndRenew() {
	echo "Checking domain '${DOMAIN}'"
	CERTFILE=/etc/letsencrypt/live/${DOMAIN}/cert.pem
	
	# Check expiration date
	if [ -f ${CERTFILE} ]; then
		d1=$(date -d "`openssl x509 -in ${CERTFILE} -text -noout | grep "Not After" | cut -c 25-`" +%s)
		d2=$(date -d "now" +%s)
		DAYSEXP=`echo \(${d1} - ${d2}\) / 86400 | bc`
		echo "Domain '${DOMAIN}' will expire in ${DAYSEXP} days"
	else
		echo "Domain '${DOMAIN}' has no cert yet"
		# Set DAYSEXP so that a cert will be requested, but no alert will be issued
		DAYSEXP=${DAYSTOALERT}
	fi
	
	
	# Check if cert needs to be renewed
	if [ ${DAYSEXP} -lt ${DAYSTORENEW} ]; then
		echo "Trying to renew..."
	
		# Pre-Test to not mess up the server if you are not answering the challenge
		TESTFILE=${RANDOM}
		URL=http://${DOMAIN}/${ACMEPATH}/${TESTFILE}
		mkdir -p ${WEBROOT}/${ACMEPATH}
		echo "test" > ${WEBROOT}/${ACMEPATH}/${TESTFILE}
	
		if curl --output /dev/null --silent --head --fail "${URL}"; then
			${LEBIN} --renew-by-default -a webroot --webroot-path ${WEBROOT} --email ${LEEMAIL} --text --agree-tos -d ${DOMAIN} auth
			APACHE_RELOAD=true
		else
			echo ""
			echo "Cannot access the pre-challenge '${URL}'"
			echo "Please add an alias to your webserver:"
			echo "'Alias /.well-known ${WEBROOT}/.well-known'"
		fi
	
		rm -f "${WEBROOT}/${ACMEPATH}/${TESTFILE}"
	else
		echo "Cert is not yet to expire"
	fi
	
	if [ ${DAYSEXP} -lt ${DAYSTOALERT} ]; then
		msg="Alert! The domain '${DOMAIN}' might have a certificate renewal problem. Please check it manually."
	
		# Send email if adress is configured
		if [ -n ${ALERTEMAIL} ]; then
			echo "${msg}" | mail -s "Let's Encrypt Automation" ${ALERTEMAIL}
		fi
	
		echo "${msg}"
	fi

	echo ""
}



# Parse the params
if [ -z $1 ]; then
	echo "Please specify one or more domains (separated by a space) as a parameter or '--renew-all' to process all existing certs."
	exit 1
fi

# Renew all domains or specific ones?
if [ $1 = "--renew-all" ]; then
	ALLDOMAINS=`ls -1 /etc/letsencrypt/live`
else
	# Array will now only contain all the domains specified via command line
	ALLDOMAINS=$*
fi

for DOMAIN in ${ALLDOMAINS}; do
    checkAndRenew
done

# Reload Apache only if necessary
if [ "${APACHE_RELOAD}" = true ]; then
	echo "Reloading Apache configuration"
	apache2ctl -t && /etc/init.d/apache2 reload
fi
