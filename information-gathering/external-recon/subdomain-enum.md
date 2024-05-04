# Subdomain Enum

## Passive

Use [amass (v3)](https://github.com/owasp-amass/amass/tree/v3), [bbot](https://github.com/blacklanternsecurity/bbot), [crt](https://github.com/cemulus/crt), [github-subdomain](https://github.com/gwen001/github-subdomains), [gitlab-subdomain](https://github.com/gwen001/gitlab-subdomains) and [subfinder](https://github.com/projectdiscovery/subfinder).

{% code title="Use amass (v3)" %}
```bash
amass enum -passive -d $DOMAIN -config ~/.config/amass/config.ini \
    -timeout 30 -o amass.txt -log amass.log &>/dev/null
```
{% endcode %}

{% code title="Use bbot" %}
```bash
# Single Target
bbot -t $DOMAIN -f subdomain-enum -rf passive -em massdns \
    -y $( [[ $PROXY == true ]] && echo "-c http_proxy=$PROXY_ADDRESS" || echo "") \
    -n bbot -o ./ -n bbot

# View Data
cat ./bbot/output.ndjson jq -r 'select(.scope_distance==0) \
    | select(.type=="DNS_NAME") | .data' | sort -u
```
{% endcode %}

{% code title="Use crt" %}
```bash
# Single Target
crt -s -l 999999 -json $DOMAIN | tee ./crt.txt

# View Data
cat crt.txt | jq -r '.[].subdomain' | sed -e 's/^\*\.//' | sort -u
```
{% endcode %}

{% code title="Use github-subdomains" %}
```sh
github-subdomains -d $DOMAIN -t $GITHUB_TOKENS \
    $( [[ $DEEP == true ]] && echo "" || echo "-k -q") \
    -o ./github-subdomains.txt
```
{% endcode %}

{% code title="Use gitlab-subdomains" %}
```bash
gitlab-subdomains -d $DOMAIN -t $GITLAB_TOKEN | tee ./gitlab-subdomains.txt

# View Data
# cat ${DOMAIN}.txt
cat ./gitlab-subdomains.txt
```
{% endcode %}

{% code title="subfinder" %}
```bash
# Single Target
subfinder -d $DOMAIN -all -v -es github \
    -o subfinder.txt \
    -proxy socks5://127.0.0.1:7890 \
    -provider-config ~/.config/subfinder/provider-config.yaml

# Multiple Targets
subfinder -l $DOMAINS -all -v -es github \
    -o subfinder.txt \
    -proxy socks5://127.0.0.1:7890 \
    -provider-config ~/.config/subfinder/provider-config.yaml
```
{% endcode %}

## Active

Use [puredns](https://github.com/d3mondev/puredns) & [tlsx](https://github.com/projectdiscovery/tlsx)

{% code title="Use puredns active" %}
```bash
puredns resolve subs_passive.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt
```
{% endcode %}

{% code title="Use tlsx active" %}
```bash
cat ./scans/subs/resolved.txt \
    | tlsx -san -cn -ro -c 1000 -silent \
    | grep \.${DOMAIN}$ > tlsx.txt

# View Data
cat tlsx.txt
```
{% endcode %}

## NOERROR

{% hint style="info" %}
If the domain name does not have a wildcard
{% endhint %}

Use [dnsx](https://github.com/projectdiscovery/dnsx).

{% code title="Use dnsx noerror" %}
```bash
# Identify Wildcard
echo error.abc.xyz.target.com \
    | dnsx -r resolvers.txt -rcode noerror,nxdomain -retry 3 -silent

# NOERROR Enumeration
dnsx -d $DOMAIN -r resolvers.txt -silent -rcode noerror \
    -w subs_wordlists.txt
    | cut -d ' ' -f 1 | tee ./dnsx.txt

# View Data
cat ./dnsx.txt
```
{% endcode %}

## Brute

Use [puredns](https://github.com/d3mondev/puredns).

{% code title="Use puredns brute" %}
```bash
DEEP=true # If your machine works well

puredns bruteforce subs_wordlists.txt $DOMAIN \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt

DEEP=false
```
{% endcode %}

## Permutation

{% hint style="info" %}
If the number of subdomains is less than 500
{% endhint %}

Use [gotator](https://github.com/Josue87/gotator) and [puredns](https://github.com/d3mondev/puredns)

```bash
gotator -sub resolved.txt -perm permutations.txt \
    -depth 1 -numbers 3 -mindup -adv -md -silent > gotator-1.txt

puredns resolve gotator-1.txt \
    -r $RESOLVERS --resolvers-trusted $RESOLVERS_TRUSTED \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns-1.txt

# Do it again
gotator -sub resolved.txt -perm puredns-1.txt \
    -depth 1 -numbers 3 -mindup -adv -md -silent > gotator-2.txt

puredns resolve gotator-2.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns-2.txt

# View Data
cat puredns-1.txt puredns-2.txt | sort -u
```

## Regex Permutations

use [regulator](https://github.com/cramppet/regulator) and [puredns](https://github.com/d3mondev/puredns).

```bash
python3 main.py -t $DOMAIN -f subdomains.txt -o regulator.txt

puredns resolve regulator.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt
```

## Recursive

### Recursive Passive

Use [dsieve](https://github.com/trickest/dsieve), [amass (v3)](https://github.com/owasp-amass/amass/tree/v3.23.3) and [puredns](https://github.com/d3mondev/puredns)

```bash
dsieve -if subdomains.txt -f 3 -top 10 > dsieve.txt

amass enum -passive -df dsieve.txt -nf subdomains.txt -timeout 30 -o amass.txt

puredns resolve amass.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt
```

### Recursive Brute

{% hint style="info" %}
If the number of subdomains is less than 500
{% endhint %}

{% hint style="info" %}
It may take more than 50 minutes
{% endhint %}

Use [dsieve](https://github.com/trickest/dsieve),  [ripgen](https://github.com/resyncgg/ripgen), [gotator](https://github.com/Josue87/gotator) and [puredns](https://github.com/d3mondev/puredns)

```bash
# First
dsieve -if subdomains.txt -f 3 -top 10 >./dsieve.txt

ripgen -d ./dsieve.txt -w subs_wordlists.txt >./ripgen.txt

puredns resolve ripgen.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns-1.txt

# Do it again
gotator -sub resolved.txt -perm puredns-1.txt \
    -depth 1 -numbers 3 -mindup -adv -md -silent > gotator.txt

puredns resolve gotator.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns-2.txt

# View Data
cat puredns-1.txt puredns-2.txt | sort -u
```

## DNS

```bash
dnsx -r resolvers_trusted.txt -a -aaaa -cname -ns -ptr -mx -soa \
    -silent -retry 3 -json \
    -o dnsx.json \
    >>dnsx.log

cat dnsx.json | jq -r 'try .a[], try .aaaa[], try .cname[], try .ns[], try .ptr[], try .mx[], try .soa[]' 2>/dev/null \
    | grep "\.${DOMAIN}$" \
    | grep -E '^((http|https):\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{1,}(\/.*)?$' \
    >dnsx.txt
cat dnsx.json | jq -r '.a[]' 2>/dev/null | sort -u \
    | hakip2host \
    | cut -d ' ' -f 3 | unfurl -u domains \
    | sed -e 's/*\.//' -e 's/\.$//' -e '/\./!d' \
    | grep "\.${DOMAIN}$" \
    >hakip2host.txt

cat dnsx.txt hakip2host.txt >dnsx_and_hakip2host.txt
puredns resolve dnsx_and_hakip2host.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch  1500000 \
    -w puredns.txt \
    &>puredns.log 2>&1

sort -u puredns.txt -o puredns.txt

# View Data
cat puredns.txt
```

## Scarping

{% hint style="info" %}
If the number of subdomains is less than 500
{% endhint %}

```bash
# First request for website data
cat subdomains.txt | httpx -follow-host-redirects -status-code \
    -threads 50 -rl 150 -timeout 10 -silent \
    -retries 2 -title -web-server -tech-detect \
    -location -no-color -json -o httpx-1.json \
    2>>"httpx-1.log" >/dev/null

# Filter domains and urls in the data
cat httpx-1.json | jq -r 'try .url' 2>/dev/null \
    | grep "\.${DOMAIN}$" \
    | grep -E '^((http|https):\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{1,}(\/.*)?$' \
    | sed "s/*.//" \
    | anew httpx-urls-1.txt \
    | unfurl -u domains \
    2>>/dev/null \
    | anew -q httpx-domains-1.txt

# Second request for website data
httpx -l httpx-urls-1.txt \
    -tls-grab -tls-probe -csp-probe -status-code \
    -threads 50 -rl 150 -timeout 10 -silent \
    -retries 2 -no-color -json \
    -o httpx-2.json \
    2>>"httpx-2.log" \
    >/dev/null

# Filter the domains and urls in the data and request the site data again
cat httpx-2.json | jq -r 'try ."tls-grab"."dns_names"[],try .csp.domains[],try .url' 2>/dev/null \
    | grep "$DOMAIN" 
    | grep -E '^((http|https):\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{1,}(\/.*)?$' \
    | sed "s/*.//" | sort -u \
    | httpx -silent \
    | anew httpx-urls-2.txt \
    | unfurl -u domains \
    2>>/dev/null \
    | anew -q httpx-domains-2.txt

# Clean httpx outptu
cat httpx-urls-1.txt httpx-urls-2.txt >httpx-urls-all.txt
cat httpx-domains-1.txt httpx-domains-2.txt > httpx-domains-all.txt

# Run katana
katana -silent -list httpx-urls-all.txt -jc -kf all -c 20 -d 2 -fs rdn \
    -o katana.txt \
    2>>katana.log \
    >/dev/null

# Clean katana outptu
sed -i '/^.\{2048\}./d' ${WRITE_PATH}/subdomains/scrap/katana.txt
cat katana.txt | unfurl -u domains 2>/dev/null \
    | grep "\.${DOMAIN}$" \
    | grep -E '^((http|https):\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{1,}(\/.*)?$' \
    >katana-domains.txt

# Clean all domains
cat httpx-domains-all.txt katana-domains.txt | sort -u \
    >httpx_and_katana_domains.txt

# Resolution
puredns resolve httpx_and_katana_domains.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt \
    &>puredns.log

sort -u puredns.txt -o puredns.txt

# View Data
cat puredns.txt
```

## Google Analytics ID

```bash
# Request for website data
analyticsrelationships -ch < httpx-urls-all.txt \
    >analyticsrelationships.txt \
    2>analyticsrelationships.log

# Clean outptu
cat analyticsrelationships.txt \
    | grep "\.$DOMAIN$\|^$DOMAIN$" \
    | grep -E '^((http|https):\/\/)?([a-zA-Z0-9]([a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{1,}(\/.*)?$' \
    | sed "s/|__ //" \
    | anew -q analyticsrelationships-domains.txt

# Resolution
puredns resolve ${WRITE_PATH}/subdomains/analytics/analyticsrelationships-domains.txt \
    -r resolvers.txt --resolvers-trusted resolvers_trusted.txt \
    -l 0 \
    --rate-limit-trusted 200 \
    --wildcard-tests 30 \
    --wildcard-batch 1500000 \
    -w puredns.txt \
    &>puredns.log

sort -u puredns.txt -o puredns.txt

# View Data
cat puredns.txt
```

## Other Technology

### DNS zone transfer

{% code title="https://github.com/darkoperator/dnsrecon" %}
```bash
dnsrecon -t axfr -d target.com
```
{% endcode %}

## Preparations

### RESOLVER

```bash
RESOLVERS_URL=https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt
RESOLVERS_TRUSTED_URL=https://raw.githubusercontent.com/trickest/resolvers/main/resolvers-trusted.txt
RESOLVERS=./resolvers.txt
RESOLVERS_TRUSTED=./resolvers_trusted.txt

iresolver -threads 200 -retry 1 -count 10000 \
    -target $RESOLVERS_URL -output $RESOLVERS
iresolver -threads 200 -retry 3 \
    -target $RESOLVERS_TRUSTED_URL -output $RESOLVERS_TRUSTED
# Can also be used directly, but it may not be accurate
wget -q -O - $RESOLVERS_URL >$RESOLVERS
wget -q -O - $RESOLVERS_TRUSTED_URL >$RESOLVERS_TRUSTED

cat $RESOLVERS | wc -l
cat $RESOLVERS_TRUSTED | wc -l
```

### DICTIONARY

```bash
wget -q -O - $SUBDOMAIN_DICT_URL > $SUBDOMAIN_DICT_FILE
wget -q -O - $SUBDOMAIN_DICT_BIG_URL > $SUBDOMAIN_DICT_BIG_FILE
wget -q -O - $PERMUTATION_DICT_URL > $PERMUTATION_DICT_FILE
```