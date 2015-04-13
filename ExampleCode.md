# Introduction #

Some examples on extending applications to communicate with Squirtle. Certainly not complete but a stepping stone to begin with.

Requires the NTLM Type 2 message in Base64 format (just copy it from the server message) or the protocol flags and server nonce.

# Python, Type 2 message #

```
import urllib, urllib2, simplejson

def squirtle_type2(self, env, connection, sesskey = '', msg2):
    NTLM_msg3 = ''
    msg2 = urllib.quote(msg2)
    auth_handler = urllib2.HTTPBasicAuthHandler()
    auth_handler.add_password(realm='Squirtle Realm',
                              uri=env['SQURL'],
                              user=env['SQUSER'],
                              passwd=env['SQPASS'])
    urlopener = urllib2.build_opener(auth_handler)
    urllib2.install_opener(urlopener)
   
    dutchieurl = "http://localhost:8080/controller/type2?key=%s&type2=%s" % (sesskey, msg2)
    try:
        res = urllib2.urlopen(dutchieurl)
    except urllib2.URLError, e:
        print "Error talking to Squirtle:", e.code, ":", e.reason
        return ''

    response = res.read()
    try:
        response = simplejson.loads(response)
    except Exception, e:
        print "Error receiving response from Squirtle:", e.reason
        return ''

    if response['status'] == 'ok':
        NTLM_msg3 = response['result']

    return NTLM_msg3
```

# C (libcurl) example #

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <curl/curl.h>

struct sqdata_in {
  size_t size;
  size_t len;
  char *data;
};

static size_t write_data(void *buffer, size_t size, size_t nmemb, void *userp) {
  struct sqdata_in *wdi = userp;

  while(wdi->len + (size * nmemb) >= wdi->size) {
    /* check for realloc failing in real code. */
    wdi->data = realloc(wdi->data, wdi->size*2);
    wdi->size*=2;
  }

  memcpy(wdi->data + wdi->len, buffer, size * nmemb);
  wdi->len+=size*nmemb;

  return size * nmemb;
}

int do_squirtle(char *sesskey, char *msg2)
{
    CURL *handle;
    CURLcode res;
    struct sqdata_in sqdata;
    struct json_object *jobj;
    struct json_tokener *tok;
    char dutchieurl[4096];
    
    memset(&sqdata, 0, sizeof(sqdata));
    
    handle = curl_easy_init();
    if(handle)
    {
        sqdata.size = 1024;
        sqdata.data = malloc(sqdata.size);
        sprintf(dutchieurl, "http://localhost:8080/controller/type2?key=%s&type2=%s", sesskey, msg2);
        curl_easy_setopt(handle, CURLOPT_USERPWD, "squirtle:eltriuqs");
        curl_easy_setopt(handle, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);
        curl_easy_setopt(handle, CURLOPT_URL, dutchieurl);
        curl_easy_setopt(handle, CURLOPT_WRITEFUNCTION, write_data);
        curl_easy_setopt(handle, CURLOPT_WRITEDATA, &sqdata);
        res = curl_easy_perform(handle);
        printf("Result: %s\n", curl_easy_strerror(res));
        curl_easy_cleanup(handle);
    } else {
        printf("Error getting CURL handle\n");
        exit(-1);
    }

    if (sqdata.len > 0) {
        // Process the data here
        printf("Result: %s\n\n", sqdata.data);        
    } else {
        printf("No data returned\n");
    }
    free(sqdata.data);
    return 0;
}
```