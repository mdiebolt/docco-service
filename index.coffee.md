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

    compile = (streamOrData, out) ->
      writeBody = (body, tempName) ->
        out.setHeader "Content-Type", "text/html"
        out.setHeader "Content-Length", body.length
        out.end replaceUuid(body, tempName, fileName)

      finish = ->
        tempName = uuid.v1()
        file = tempName + extension

        fs.writeFile "#{file}", data, (err) ->
          unless err
            exec "node_modules/.bin/docco #{file}", ->
              fs.readFile "docs/#{tempName}.coffee.html", "utf8", (err, body) ->
                writeBody(body, tempName)

Clean up temporary files

              fs.unlink file
              fs.unlink "docs/#{tempName}.coffee.html"

      if typeof streamOrData is "string"
        data = streamOrData
        finish()
      else
        stream = streamOrData
        stream.setEncoding "utf8"
        stream.on "data", (chunk) ->
          data += chunk
        stream.on "end", finish

Get from a url or tell people how to use.
Compile from a url on the web.

    app.get "/", (request, response) ->
      if url = request.query.url
        setExtension(url)
        http.get url, (error, resp, body) ->
          compile(body, response)
      else # TODO present a nicer instructions view
        response.send """
          <h1 style='color:red;'>
            GET /?url=http://a-site.com/my-docco-file.coffee
          </h1>
        """

Serve up the compiled Docco output

    app.use express.static "#{__dirname}/docs"

Start up the server

    port = process.env.PORT || 5000
    app.listen port, ->
      console.log "Listening on #{port}"