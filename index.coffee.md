Docco your code as a service.

    Docco = require "docco"

    fs = require "fs"
    http = require "request"
    uuid = require "uuid"
    express = require "express"

    {exec} = require "child_process"

    app = express()
    app.use express.logger()

    extension = null
    fileName = null

Figure out the file extension based on the request URL

    setExtension = (str) ->
      [_..., fileName] = str.split("/")
      [baseName, extensions...] = fileName.split(".")

      fileName = baseName
      extension = "." + extensions.join(".")

Remove the file UUID from the response body with the actual name of the file

    replaceUuid = (str, tempName, fileName) ->
      regexp = RegExp(tempName, "g")
      str.replace(regexp, fileName)

Deal with streamed data

    processStream = (stream, cb) ->
      data = ""

      stream.setEncoding "utf8"
      stream.on "data", (chunk) ->
        data += chunk
      stream.on "end", cb(data)

Use Docco to document the code from the passed in url

    compile = (data, out) ->
      writeBody = (body, tempName) ->
        content = replaceUuid(body, tempName, fileName)

        out.setHeader "Content-Type", "text/html"
        out.setHeader "Content-Length", content.length
        out.end content

      tempName = uuid.v1()
      file = tempName + extension

      fs.writeFileSync "#{file}", data

      exec "node_modules/.bin/docco #{file}", ->
        body = fs.readFileSync "docs/#{tempName}.coffee.html", "utf8"
        writeBody(body, tempName)

Clean up temporary files

        fs.unlinkSync file
        fs.unlinkSync "docs/#{tempName}.coffee.html"

Get from a url or tell people how to use.
Compile from a url on the web.

    app.get "/", (request, response) ->
      if url = request.query.url
        setExtension(url)
        http.get url, (error, resp, body) ->
          compile(body, response)
      else
        response.send """
          <div style='width:450px; height:150px; background-color: #dedede; border-radius: 4px; font-family: "Helvetica Neue", Arial, sans-serif; position:absolute; left:0; right:0; top:0; bottom:0; margin:auto; padding: 1em 2em;'>
            <p>Use the <b>url</b> query string parameter to point this webservice to the source of a Docco compatible documented file.</p>
            <p>Docco will be run against the source, and the resulting html will be served to the browser.</p>
            <p>Usage: GET /?url=http://a-site.com/my-docco-file.coffee</p>
          </div>
        """

Serve up the compiled Docco output

    app.use express.static "#{__dirname}/docs"

Start up the server

    port = process.env.PORT || 5000
    app.listen port, ->
      console.log "Listening on #{port}"