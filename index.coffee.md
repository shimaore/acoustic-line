    doctypes =
      'xml': '<?xml version="1.0" encoding="utf-8" ?>\n'

    elements =
      # requiring a closing tag
      regular: 'document section configuration settings modules list global_settings profiles profile context extension condition mappings language macro match input macros'

      # self-closing
      void: 'param load node action map'

    # Create a unique list of element names merging the desired groups.
    merge_elements = (args...) ->
      result = []
      for a in args
        for element in elements[a].split ' '
          result.push element unless element in result
      result

    decamelcaseify = (name) ->
      name.replace /[A-Z]/g, ($,$1) -> "-#{$1.toLowerCase()}"

    class AcousticLine
      constructor: ->
        @out = null

      resetBuffer: (xml=null) ->
        previous = @out
        @out = xml
        return previous

      render: (template,args...) ->
        previous = @resetBuffer ''
        try
          template args...
        finally
          result = @resetBuffer previous
        return result

      renderable: (template) ->
        acoustic = @
        return (args...) ->
          if acoustic.out is null
            acoustic.out = ''
            try
              template.apply this, args
            finally
              result = acoustic.resetBuffer()
            return result
          else
            template.apply this, args

      renderAttr: (name, value) ->
        if not value?
          return " #{name}"

        return " #{name}=#{@quote @escape value.toString()}"

      attrOrder: ['application','function','data','name','value','field','expression','description']
      renderAttrs: (obj) ->
        result = ''

        # render explicitely ordered attributes first
        for name in @attrOrder when name of obj
          result += @renderAttr name, obj[name]
          delete obj[name]

        # the unordered attrs
        for name, value of obj
          result += @renderAttr name, value

        return result

      renderContents: (contents) ->
        if not contents?
          return
        else if typeof contents is 'function'
          result = contents.call this
          @text result if typeof result in ['string','number','boolean']
        else
          @text contents

For some tags we allow the user to specify one or more native values directly in a syntax that "does what you expect".

      valuedArgs:
        action: ['application','data']
        'anti-action': ['application','data']
        condition: ['field','expression']
        configuration: ['name']
        context: ['name']
        extension: ['name']
        input: ['pattern','break_on_match']
        language: ['name','sound-path']
        load: ['module']
        macro: ['name']
        param: ['name','value']
        section: ['name','description']

      normalizeArgs: (tag,args) ->
        attrs = {}
        contents = null
        for arg, index in args when arg?
          switch typeof arg
            when 'string', 'number', 'boolean'
              attr_name = @valuedArgs[tag]?[index]
              if attr_name?
                attrs[attr_name] = arg
              else
                contents = arg
            when 'function'
              contents = arg
            when 'object'
              if arg.constructor == Object
                for own attr_name, value of arg
                  attrs[decamelcaseify attr_name] = value
              else
                contents = arg
            else
              contents = arg

        return {attrs,contents}

      tag: (tag, args...) ->
        tag = decamelcaseify tag
        {attrs,contents} = @normalizeArgs tag, args
        @raw "<#{tag}#{@renderAttrs attrs}>\n"
        @renderContents contents
        @raw "</#{tag}>\n"

      selfClosingTag: (tag, args...) ->
        tag = decamelcaseify tag
        {attrs,contents} = @normalizeArgs tag, args
        if contents
          throw new Error "AcousticLine: <#{tag}/> must not have content. Attempted to nest #{contents}"
        @raw "<#{tag}#{@renderAttrs attrs}/>\n"

      comment: (text) ->
        @raw "<!-- #{@escape text} -->"

      doctype: (type='xml') ->
        @raw doctypes[type]

      text: (s) ->
        unless @out?
          throw new Error "AcousticLine: can't call a tag function outside a rendering context"
        @out += s? and @escape(s.toString()) or ''
        null

      raw: (s) ->
        return unless s?
        @out += s
        null

      #
      # Special case in FreeSwitch XML syntax
      #
      network_lists: (args...) ->
        @tag 'network-lists', args...

      anti_action: (args...) ->
        @selfClosingTag 'anti-action', args...

      #
      # Filters
      #

      escape: (text) ->
        text.toString()
        .replace /&/g, '&amp;'
        .replace /</, '&lt;'
        .replace />/, '&gt;'
        .replace /"/g, '&quot;'
        .replace /'/g, '&apos;'

      quote: (value) ->
        "\"#{value}\""

      #
      # Plugins
      #
      use: (plugin) ->
        plugin this

      #
      # Binding
      #
      tags: ->
        bound = {}

        boundMethodNames = [].concat(
          'comment doctype escape normalizeArgs raw render renderable text use network_lists anti_action'.split ' '
          merge_elements 'regular', 'void'
        )
        for method in boundMethodNames
          do (method) =>
            bound[method] = (args...) => @[method].apply this, args

        return bound

    for tagName in merge_elements 'regular'
      do (tagName) ->
        AcousticLine::[tagName] = (args...) -> @tag tagName, args...

    for tagName in merge_elements 'void'
      do (tagName) ->
        AcousticLine::[tagName] = (args...) -> @selfClosingTag tagName, args...

    module.exports = new AcousticLine().tags()
    module.exports.AcousticLine = AcousticLine
