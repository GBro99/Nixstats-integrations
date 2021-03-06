Add Nixstats notifications via a new WebHook in Rocket.Chat

1. In Rocket.Chat go to "Administration"->"Integrations" and create "New Integration"
2. Choose Outgoing WebHook
3. Select **Message Sent** as Event trigger
3. Enter **ns** as trigger word
4. Enter **https://api.eu.nixstats.com/v1/** as URLs
5. Avatar URL **https://nixstats.com/images/favicon.png**
6. **Token**, this is your nixstats API token, [create an API key](https://nixstats.com/settings/api).
7. Script Enabled set to **True**

Paste this in javascript in the "Script" textarea on Rocket.Chat webhook settings


```/* exported Script */
/* globals Store */

class Script {
  prepare_outgoing_request({ request }) {
    let match; 
   
    match = request.data.text.match(/^ns servers\s(ls|list)\s*(.*)?$/); 
    if (match) { 
      let u = request.url + 'servers?perpage=99&token='+request.data.token; 
      return {
        url: u,
        headers: request.headers,
        method: 'GET'
      };
    }
    
    match = request.data.text.match(/^ns graphs\s(.*)?$/);
    if (match) { 
      var matched = false; 
      var options;
      var serverrequest = HTTP('GET', request.url + 'servers?perpage=99&token='+request.data.token, options);
      var serverlist = []
      JSON.parse(serverrequest.result.content).servers.forEach(function(pr) {
        serverlist.push({'name': pr.name, 'id': pr.id});
      });
        
      serverlist.forEach(function(serv) {
        if(serv.id == match[1])
        {
          matched = serv.id;
        }
        if(serv.name == match[1])
        {
          matched = serv.id;
        }
      });
               
      if(!matched){
        return {
          message: {
            text: 'Server not found.'
          }
        }; 
      }
      else
      {
        let u = request.url + 'server/'+matched+'?charts=yes&token='+request.data.token; 
        return {
          url: u,
          headers: request.headers,
          method: 'GET'
        };
      }
    }
     
    match = request.data.text.match(/^help$/);
    if (match) { 
      return {
        message: {
          text: [
            '**Nixstats commands**',
            '```',
              '  ns servers ls|list',
              '  ns graphs serverid',
            '```'
          ].join('\n')
        }
      };
    }
  }

  process_outgoing_response({ request, response }) {
    var text = [];
    var attach = []; 
    if(response.content.charts)
    {
      response.content.charts.forEach(function(pr) {
        attach.push({
           "color": "#000000",
           "text": pr.title+" on "+response.content.name,
           "image_url": pr.url,
         });
      });
        text.push('Performance of '+response.content.name);
    }
    else
    {
      text.push('```'); 
      response.content.servers.forEach(function(pr) {
        text.push(''+pr.id+"\t "+pr.last_data.load.replace(",",",\t")+"\t"+pr.name+'');
      });
      text.push('```'); 
    }
    return {
      content: {
        text: text.join('\n'),
        attachments: attach,
        parseUrls: false
      }
    };
  }
}
```


After saving the data you can use the following commands to retrieve data.

`ns servers list` to list your servers with their ID's and load average.

<img src="serverlist.png" data-canonical-src="graphs.png" width="400" />

`ns graphs [serverid]` to retrieve a graph of Memory, Network, Load average and Disk usage of the specified server.

<img src="graphs.png" data-canonical-src="graphs.png" width="400" />
