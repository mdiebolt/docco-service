Docco your code as a service.

    docco = require("docco")
    http = require("request")
    express = require("express")

    app = express()
    app.use express.logger()

    compile = (streamOrData, out) ->
      finish = ->
        try
          body = Docco.run(data)
          console.log body
          out.setHeader('Content-Type', 'text/html')
          out.setHeader('Content-Length', body.length)
          out.end(body)
        catch err
          console.log err
          out.status(400)

      if typeof streamOrData is "string"
        data = streamOrData
        finish()
      else
        stream = streamOrData
        stream.setEncoding('utf8')
        stream.on 'data', (chunk) ->
          data += chunk
        stream.on 'end', finish

Get from a url or tell people how to use.

    app.get '/', (request, response) ->
      if url = request.query.url
        console.log "URL: #{url}"
        http.get url, (error, resp, body) ->
          compile(body, response)
      else
        message = """
          GET /?url=http://a-site.com/my.coffee.md
          POST CoffeeScript source to /
        """

        response.send(message)

Compile from a url on the web.

Docco the file body and return the styled html documentation.

    app.post '/', compile

Listen up server!

    port = process.env.PORT || 5000
    app.listen port, ->
      console.log("Listening on " + port)