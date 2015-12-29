    doctypes =
      'xml': '<?xml version="1.0" encoding="utf-8" ?>\n'

The (complete?) list of elements might be obtained by running `rgrep switch_xml_child | perl -ane ' /"([^"]+)"/ && print "$1\n"; ' | sort -u` from within the FreeSwitch source. (That gives me 219 different element types.)
For now I only list the ones I use most of the time, but feel free to open a PR for any other, or create a plugin that extends `acoustic-line` to support these; in the meantime the `tag` method can be used to create any that's needed.

    elements =
      # requiring a closing tag
      regular: 'document section configuration settings modules list global_settings profiles profile context extension condition mappings language macro match input macros params gateway gateways fifos fifo'

      # self-closing
      void: 'param load node action map'

Create a unique list of element names merging the desired groups.

    merge_elements = (args...) ->
      result = []
      for a in args
        for element in elements[a].split ' '
          result.push element unless element in result
      result

There's no issue with attribute names that contain underscore as they may easily be expressed as-such in JavaScript.
However for the few cases where this is needed (`digit-len`, `duration-header`, ..) you may use CamelCase to express a dash (`digitLen`, `durationHeader`, ...).
The (complete?) list of attributes might be obtained by running `rgrep switch_xml_attr_soft src/ | perl -ane ' /"([^"]+)"/ && print "$1\n"' | sort -u` from within the FreeSwitch source.

    decamelcaseify = (name) ->
      name.replace /[A-Z]/g, ($,$1) -> "-#{$1.toLowerCase()}"

AcousticLine
============

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
        document: ['type']
        extension: ['name']
        gateway: ['name']
        input: ['pattern','break_on_match']
        language: ['name','sound-path']
        load: ['module']
        macro: ['name']
        map: ['name','value']
        param: ['name','value']
        profile: ['name']
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
                if contents? then throw new Error "AcousticLine: <#{tag}/> attempted to set content twice."
                contents = arg
            when 'function'
              if contents? then throw new Error "AcousticLine: <#{tag}/> attempted to set content twice."
              contents = arg
            when 'object'
              if arg.constructor == Object
                for own attr_name, value of arg
                  attrs[decamelcaseify attr_name] = value
              else
                if contents? then throw new Error "AcousticLine: <#{tag}/> attempted to set content twice."
                contents = arg
            else
              if contents? then throw new Error "AcousticLine: <#{tag}/> attempted to set content twice."
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

Special case in FreeSwitch XML syntax
-------------------------------------

These are tags I use frequently which do not follow the generic syntax.
These could also be created by saying things like
```
{AcousticLine} = require 'acoustic-line'
AcousticLine::network_lists = (args...) -> @tag 'network-lists', args...
```

      network_lists: (args...) ->
        @tag 'network-lists', args...

      anti_action: (args...) ->
        @selfClosingTag 'anti-action', args...

Filters
-------

      escape: (text) ->
        text.toString()
        .replace /&/g, '&amp;'
        .replace /</, '&lt;'
        .replace />/, '&gt;'
        .replace /"/g, '&quot;'
        .replace /'/g, '&apos;'

      quote: (value) ->
        "\"#{value}\""

Plugins
-------

      use: (plugin) ->
        plugin this

Binding
-------

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

Automatically create elements listed in `regular` and `void`.

    for tagName in merge_elements 'regular'
      do (tagName) ->
        AcousticLine::[tagName] = (args...) -> @tag tagName, args...

    for tagName in merge_elements 'void'
      do (tagName) ->
        AcousticLine::[tagName] = (args...) -> @selfClosingTag tagName, args...

    module.exports = new AcousticLine().tags()
    module.exports.AcousticLine = AcousticLine
