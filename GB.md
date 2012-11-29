GRIDBLAZE is a supercharged object storage platform for user generated content like images and files. With over 15 storage nodes spread globally and smart routing, user uploaded files are uploaded at 4 times the speed to storage endpoints close to your users. In reverse, downloads are made blazingly fast as well, as data is already stored in close proximity to the requester.

The storage platform supports ingest and access through regular HTTP via POST and GET commands respectively. An alternative method to interact with storage will be through the OpenStack compatible API or web based file manager.

## Provisioning the add-on

<div class="callout" markdown="1">
A list of all plans available can be found [here](http://addons.heroku.com/gridblaze).
</div>

To use GRIDBLAZE on Heroku, install the GRIDBLAZE add-on:

    :::term
    $ heroku addons:add gridblaze
    -----> Adding GRIDBLAZE to sharp-mountain-4005... done, v112 (free)

To begin using the storage platform you will need to get an id that is automatically generated for each account. Each id will also have a corresponding key which should be kept secret.

    :::term
    $ heroku config | grep GRIDBLAZE
    GRIDBLAZE_APPID:                h123i743
    GRIDBLAZE_APPKEY:               98236498c027492910jal9777

These values will be used by your app at run time to generate a signature to authenticate uploads and access.

## Direct uploads

Direct uploading uses a simple HTTP form post to upload files from users directly to the GRIDBLAZE storage platform, bypassing your application dynos on Heroku. Files will automatically be routed to the closest storage node to the user and kept in that location for future access.

### Create upload signature

GRIDBLAZE requires that you generate a unique signature for every submitted upload. The signature is composed of the GRIDBLAZE app id and key and the following user-specified fields:

<table>
  <tr>
    <th>Name</th>
    <th>Variable</th>
  </tr>
  <tr>
    <td>return_url</td>
    <td style="text-align: left">The URL web hook that GRIDBLAZE will call to confirm the successful upload and push the URL to access the file. It should be a full URL including "http://"</td>
  </tr>
  <tr>
    <td>datetime</td>
    <td style="text-align: left">The unix epoch formatted datetime when the page is generated as encoded in the signature. Note the upload has to happen within 24hrs of this date/time.</td>
  </tr>
</table>

Create the signature as a hash of these fields, appended together and delimited by a `.`. Using Ruby as an example you can create a signature like this:

    :::ruby
    message = "#{ENV['GRIDBLAZE_APPID']}.#{ENV['GRIDBLAZE_APPKEY']}.#{return_url}.#{datetime}"
    signature = Digest::SHA256.hexdigest(message)

### Server Side Code

On top of the signature, your code will need to generate a few other variables that will need to be sent to the GRIDBLAZE system after the user POSTs the file.

    :::ruby
    appid = ENV['GRIDBLAZE_APPID']
    secret_key = ENV['GRIDBLAZE_APPKEY']
    return_url = 'http://www.mydomain.tld' # The URL web hook that GRIDBLAZE will call to confirm a successful upload and push the access information for file. It should be a full URL including "http://"
    directory = '/' #The directory to place the file. Eg,/mydirectory/
    meta = "{'name': 'myfile', 'type': 'just a file'}" #Meta text data 
    option = 'default'
    enable_auth = 'no'

    # Option include autogen, replace, reject, default. 
    upload_url = "http://upload.gridblaze.com" # Fixed, do not change 	
    datetime = Time.now.to_i # Epoch time format at the generation of the signature
    signature = Digest::SHA256.hexdigest(appid + secret_key + return_url + directory + datetime.to_s() + option + enable_auth + meta)


### HTML form

<div class="callout" markdown="1">
The form action must be set as `upload.gridblaze.com`.
</div>

The resulting HTML form generator should look like this.

    :::html
    <form action='http://upload.gridblaze.com' enctype='multipart/form-data' method='post'>
		<input type="file" name="file">		
        <input id="enable_auth" type="hidden"  name="enable_auth" value='<%= enable_auth %>' />
		<input id="appid"       type="hidden"  name="appid"       value='<%= appid %>' />
        <input id="return_url"  type="hidden"  name="return_url"  value='<%= return_url %>' />
        <input id="directory"   type="hidden"  name="directory"   value='<%= directory %>' />
        <input id="options"     type="hidden"  name="options"     value='<%= options %>' />
        <input id="meta"        type="hidden"  name="meta"        value='<%= URI::encode(meta) %>' />
        <input id="datetime"    type="hidden"  name="datetime"    value='<%= datetime %>' />
        <input id="signature"   type="hidden"  name="signature"   value='<%= signature; %>' />		
		<input type="submit" name="upload" value="submit">
    </form>

### Upload callback

After a user uploads a file to the storage network, GRIDBLAZE will call the return_url that you provided with the URL to access the file, the file size and any other form attributes that you sent along with the form post.

<p class="note" markdown="1">
Form fields not specific to GRIDBLAZE will get resubmitted back to your `return_url` location, allowing GRIDBLAZE to handle the file submission and your application to handle the other aspects of the form in a way that provides seamless user experience.
</p>

Files uploaded to the GRIDBLAZE storage network will have URLs resembling the following format:

    http://g.csn.io/us3/e640f85c/myfile.jpg

Here's an example code of the return_url

    :::html
    post '/uploadSuccess' do
        request.body.read # this will display the data
    end

## OpenStack API

GRIDBLAZE is built on top of the [OpenStack Swift API](http://www.openstack.org/software/openstack-storage/). Connecting to the API is done through the following host.

    :::html
    https://api.gridblaze.com/v1

You can find details on [GRIDBLAZE's OpenStack implementation here](http://developer.gridblaze.com/api_documentation.php).

Connecting to api.gridblaze.com will route requests to the closest API node. Unlike other APIs, our API servers are distributed to allow for lower latency and automated PUT routing to the closest node.

## Managing Storage

Managing and listing files stored on the GRIDBLAZE network can be done through the following means:

### Web Based Object Manager

This is the simplest method and would only require you to login to the GRIDBLAZE command dashboard and select "Object Manager". You will be able to upload/list/delete/rename all files stored on the GRIDBLAZE network associated with your app key. Files might be in different storage nodes but in the object manager, you will view all files in a single "disk view".

### API

Use command line or GUI tools like Cyberduck to connect to the storage network using the OpenStack API standards.

    :::html
    https://api.gridblaze.com/v1

### WebDAV

This features is still under development however you are welcome to Alpha test it by connecting to our WebDAV server on:

    :::html
    https://dav.gridblaze.com

Current issues are that of latency and speed which we are working to solve since WebDAV is not a very robust standard.

### De-provisioning or Removal

To remove GRIDBLAZE and stop using the storage service simply call the following command.

    :::html
    $ heroku addons:remove gridblaze

## Support

All GRIDBLAZE support and runtime issues should be submitted via on of the [Heroku Support channels](support-channels).