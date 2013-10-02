Docco your code as a service.

    fs = require "fs"
    http = require "request"
    express = require "express"

    {exec} = require "child_process"

    app = express()
    app.use express.logger()
    app.use express.static "#{__dirname}/docs"

    fileName = null
    baseName = null

Determine is the file is CoffeeScript with embedded Markdown

    hasMarkdownExtension = (str) ->
      str.indexOf(".md") > 0

Figure out the file extension based on the request URL

    setFileName = (str) ->
      [_..., fileName] = str.split("/")
      [baseName, extensions...] = fileName.split(".")

      fileName = fileName
      baseName = baseName

      return fileName

Use Docco to document the code from the passed in url

    compile = (data, out) ->
      writeBody = (body) ->
        out.setHeader "Content-Type", "text/html"
        out.setHeader "Content-Length", body.length
        out.end body

      fs.writeFileSync "docs/#{fileName}", data

      sourceFilePath = "docs/#{fileName}"

      compiledDocsPath = "docs/#{basename}.coffee.html"
      compiledDocsPath.replace(".coffee", "") unless hasMarkdownExtension(compiledDocsPath)

      exec "node_modules/.bin/docco #{sourceFilePath}", ->
        body = fs.readFileSync compiledDocsPath, "utf8"
        writeBody(body)

Clean up temporary files

        fs.unlinkSync sourceFilePath
        fs.unlinkSync compiledDocsPath

Get from a url or tell people how to use.
Compile from a url on the web.

    app.get "/", (request, response) ->
      if url = request.query.url
        setFileName(url)
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

Start up the server

    port = process.env.PORT || 5000
    app.listen port, ->
      console.log "Listening on #{port}"