Docco your code as a service.

    fs = require "fs"
    rr = require "rimraf"
    http = require "request"
    uuid = require "uuid"
    express = require "express"

    {exec} = require "child_process"

    app = express()
    app.use express.logger()
    app.use express.static "#{__dirname}/docs"

    fileName = null
    baseName = null

Determine is the file is CoffeeScript with embedded Markdown. We need to special case this since Docco compiles that into `<file_name>.coffee.html`

    hasMarkdownExtension = (str) ->
      str.indexOf(".md") > 0

Path to the source code that is being Docco'd

    sourcePath = (fileName, uuid) ->
      "docs/#{uuid}/#{fileName}"

Prevent collisions by outputting docs into a directory named after our uuid.

    outputPath = (uuid) ->
      "docs/#{uuid}"

Output path of the file Docco compiles. Replace the `.coffee.html` that Docco adds if we are dealing with a `.coffee.md` file.

    compiledFileName = (fileName, uuid) ->
      path = "docs/#{uuid}/#{baseName}.coffee.html"
      path = path.replace(".coffee", "") unless hasMarkdownExtension(fileName)
      path

Parse and set `fileName` and `baseName` based on the request URL.

    setFileInfo = (str) ->
      [_..., fileName] = str.split "/"
      [baseName, extensions...] = fileName.split "."

      fileName = fileName
      baseName = baseName

      return fileName

Use Docco to document the code from the passed in url

    compile = (data, out) ->
      writeBody = (body) ->
        out.setHeader "Content-Type", "text/html"
        out.setHeader "Content-Length", body.length
        out.end body

      id = uuid.v1()

      sourceFilePath = sourcePath(fileName, id)
      output = outputPath(id)
      compiledFile = compiledFileName(fileName, id)

      fs.mkdirSync output
      fs.writeFileSync sourceFilePath, data

Run Docco on the copy of our source code that we saved locally.

      exec "node_modules/.bin/docco #{sourceFilePath} -o #{output}", ->
        body = fs.readFileSync compiledFile, "utf8"
        writeBody(body)

Clean up temporary files.

        fs.unlinkSync sourceFilePath
        rr.sync "docs/#{id}"

Get from a url or tell people how to use the service.

    app.get "/", (request, response) ->
      if url = request.query.url
        setFileInfo(url)
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

Start up the server.

    port = process.env.PORT || 5000
    app.listen port, ->
      console.log "Listening on #{port}"