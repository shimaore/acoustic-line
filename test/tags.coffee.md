    {expect} = require 'chai'

    describe 'Tags', ->

      it 'doctype', ->
        {render,doctype} = require '../index.coffee.md'
        expect render -> doctype()
        .to.equal '<?xml version="1.0" encoding="utf-8" ?>'

      it 'simple document', ->
        {render,doctype,document,section,configuration,settings,param,modules,load,network_lists,list,node,global_settings,profiles,profile} = require '../index.coffee.md'
        expect render ->
          doctype()
          document type:'freeswitch/xml', ->
            section name:'configuration', ->
              configuration name:'switch.conf', ->
                settings ->
                  param name:'switchname', value:'my-switch'
                  param name:'loglevel', value:'debug'
              configuration name:'modules.conf', ->
                modules ->
                  load module:'mod_commands'
                  load module:'mod_dptools'
              configuration name:'acl.conf', ->
                network_lists ->
                  list name:'local', default:'deny', ->
                    node type:'allow', cidr:'127.0.0.0/8'
                    node type:'allow', cidr:'192.168.0.0/16'
                  list name:'rfc5737', default:'deny', ->
                    for cidr in ['192.0.2.0/24','198.51.100.0/24','203.0.113.0/24']
                      node type:'allow', cidr:cidr
              configuration name:'sofia.conf', ->
                global_settings ->
                  param name:'log-level', value:1
                  param name:'debug-presence', value:0
                profiles ->
                  profile name:'example', ->
                    settings ->
                      param name:'debug', value: 2
                      param name:'sip-trace', value:true
                      param name:'sip-port', value:5060
                      param name:'username', value:'example freeswitch'
        .to.equal '''
          <?xml version="1.0" encoding="utf-8" ?>
          <document type="freeswitch/xml">
            <section name="configuration">
              <configuration name="switch.conf">
                <settings>
                  <param name="switchname" value="my-switch"/>
                  <param name="loglevel" value="debug"/>
                </settings>
              </configuration>
              <configuration name="modules.conf">
                <modules>
                  <load module="mod_commands"/>
                  <load module="mod_dptools"/>
                </modules>
              </configuration>
              <configuration name="acl.conf">
                <network-lists>
                  <list name="local" default="deny">
                    <node type="allow" cidr="127.0.0.0/8"/>
                    <node type="allow" cidr="192.168.0.0/16"/>
                  </list>
                  <list name="rfc5737" default="deny">
                    <node type="allow" cidr="192.0.2.0/24"/>
                    <node type="allow" cidr="198.51.100.0/24"/>
                    <node type="allow" cidr="203.0.113.0/24"/>
                  </list>
                </network-lists>
              </configuration>
              <configuration name="sofia.conf">
                <global_settings>
                  <param name="log-level" value="1"/>
                  <param name="debug-presence" value="0"/>
                </global_settings>
                <profiles>
                  <profile name="example">
                    <settings>
                      <param name="debug" value="2"/>
                      <param name="sip-trace" value="true"/>
                      <param name="sip-port" value="5060"/>
                      <param name="username" value="example freeswitch"/>
                    </settings>
                  </profile>
                </profiles>
              </configuration>
            </section>
          </document>
        '''.replace /\n */g, ''
