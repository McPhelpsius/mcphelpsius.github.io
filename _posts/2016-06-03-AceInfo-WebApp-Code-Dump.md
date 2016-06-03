---
title: AceInfo Code Dump
layout: post
---

# AceInfo Code Dump

I took a job with AceInfo, a contractor who I doing weather forecasting work. This is the most interesting work I've done to date,
probably because I got in a little above my skill level. It was a lot of fun though. Below is a bunch of code I did on a project using
OpenLayers 3 and ESRI Maps. Enjoy!

#### I've excluded content I didn't create or at least work on, so the context is a bit skewed.

## Map Object

    /* This is the overarching Map object that contains OpenLayers Extents, an OpenLayers Map, 
    OpenLayers layers/layerGroups,and our custom layerHandler. This allows them all to be accessible to ecah other with multiple maps
    in the app that should cross over each other. */

    var Map = function (options) {
        this.mapDeferred = $.Deferred();
        this.options = {
            target: Math.random(),
            controls: {
                attribution: true,
                zoom: true
            },
            playback: true,
            layers: ['background']
        };
        $.extend(true, this.options, options);
        this.map = new ol.Map({
            view: new ol.View({
                projection: 'EPSG:3857',
                minZoom: 3,
                zoom:6
            }),
            controls: ol.control.defaults(this.options.controls)
        });
        this.extent = new ExtentPrototype(this.map);
        this.handler = new Handler({
            playback: this.options.playback
        });
        this.map.addLayer(this.handler.layer);
        this.addLayers();
        this.attachEvents();
    };

    Map.prototype.constructor = Map;

    Map.prototype.appendMapData = function(data, extent) {
        return $.extend(true, {}, data, {
            map: {
                target: this.options.target || null,
                ready: this.map.getTargetElement() ? true : false,
                size: this.map.getSize() || null,
                extent: extent || this.extent.getExtent() || null
            }
        });
    };

    Map.prototype.addLayers = function() {
        for (var x = 0; x < this.options.layers.length; x++) {
            this.updateMap('map:update', {layer: this.options.layers[x]});
        }
    };

    Map.prototype.updateMap = function(evt, data) {
        if (data.constructor !== Array) {
            data = [data];
        }
        var that = this;
        $.when(this.extent.getDeferred()).done(function(extent) {
            $(window).trigger('ui:loopControls:detach');
            for(var x = 0; x < data.length; x++){
                console.log("Updating " + that.options.target + " for " + data[x].layer);
                data[x] = that.appendMapData(data[x], extent);
            }

            that.handler.updateMap(data);
        });
    };

    Map.prototype.attachEvents = function () {
        var that = this;

    // "once" the map is rendered, add listeners (moveend fires when center is set on map load)
        this.map.once("moveend", function(){

            that.map.on('moveend', function() {
                log.info("Updating " + that.options.target + " layer: pan");
                that.handler.updateActiveLayers(that.appendMapData());
            });
            // Zoom in or out
            that.map.getView().on('change:resolution', function() {
                log.info("Updating " + that.options.target + " layer: zoom");
                that.handler.updateActiveLayers(that.appendMapData());
            });
            // Store values in Local Storage
            that.map.getView().on('propertychange', function(view) {
                switch (view.key) {
                    case 'resolution':
                        store.set(that.options.target + "mapZoom", this.getZoom() );
                        break;
                    case 'center':
                        var centerLonLat = ol.proj.toLonLat(that.map.getView().getCenter(), 'EPSG:900913');
                        store.set(that.options.target + "mapCenter", centerLonLat );
                        break;
                }
            });

        });
    };
    
    
    
## Layer Handler
    
    /* In this page we created the Map's Handler to handle the layers on the map that would be switched active and hidden. 
    Here we would create all the layers for the map, and create a reference to them. This would allow us to toggle which layers would be 
    showing on the map, based on layer data. We made it able to show multiple layers.

    //// As you can tell, the naming conventions are pretty on point.
    // "refName"" is a reference to the specific layerHandler and layer in it that we are creating. We have multiple maps in th app,
    // so this in necessary.
    // "classPath" is the path to the specific file where the object for the layer is in the file structure. Obviously 
    // this is for instantiating the instance of the layer.
    // "data" holds the information for the string and other data passed down from the route and Map

	Handler.prototype.createLayer = function(refName, classPath, data) {
        var that = this;
        console.log(data.map.target + ":", "Creating layer instance for " + refName + " @" + classPath)

        require([classPath], function(Layer){
            var layer = new Layer(data);
            layer.layersCreated.done(function() {
                that.group.getLayers().setAt(that.layerRef[refName].position, layer.layer);
                that.layerRef[refName].instance = layer;
                that.layerRef[refName].ready.resolve(refName);
            });
        });
        return that.layerRef[refName].ready.promise();
    };
    
    // createLayers was creating the layers for the map, and showLayers should be self explanatory
    // This is from a button click event where we want to toggle of one layer and show another one.
    // We hide all layers that are visible unless they have a "persistent" property that keeps it from ever being hidden, such
    // as the background layer

    Handler.prototype.showLayers = function (layers) {
        layers = layers || {};
        for (var name in this.layerRef) {
            var that = this;
            this.layerRef[name].ready.done(function(refName){
                if (that.layerRef[refName].instance.get('persistent') === false){
                    if (typeof layers[refName] !== 'undefined') {
                        if(typeof that.activeLayers[refName] === 'undefined'){
                            that.activeLayers[refName] = layers[refName];
                            that.layerRef[refName].instance.showLayer(true);
                            that.attachController(refName);
                        }
                        that.updateLayer(refName, layers[refName]);
                    } else {
                        that.layerRef[refName].instance.showLayer(false);
                        delete that.activeLayers[refName];
                    }
                }
            });
        }
    };

    // There is a single controls element on the page that allows for loop, moving forwaard one, and moving backwards one
    // The weather radar data has looping weather with time, so this allows for people to play and stop that loop on demand.
    // This function allows each layer to attach to these button clicks when they are shown, and unassign hidden layers.
    Handler.prototype.attachController = function (refName) {
        var layer = this.layerRef[refName].instance;
        if (typeof this.layerRef[refName].instance !== 'undefined'
            && layer.hasOwnProperty('loopControls')
            && layer.loopControls === true
            && this.options.playback) {

            $(window).trigger('ui:loopControls:attach', {
                refName: refName,
                play: function(){
                    layer.loop(true);
                },
                stop: function(){
                    layer.loop(false);
                },
                forward: function(step){
                    layer.loop(false);
                    layer.cycleLayer(-1); // eventually pass step when UI allows
                },
                back: function(step){
                    layer.loop(false);
                    layer.cycleLayer(1); // eventually pass step when UI allows
                }
            });
        }
    };

    // called with name and data
    // called with no prarams, use active
    Handler.prototype.showLayer = function (refname, data) {
        var layers = {};
        layers[refname] = data;
        this.showLayers(layers);
    };

    Handler.prototype.updateActiveLayers = function(data){
        for (var name in this.activeLayers) {
            $.extend(true, this.activeLayers[name], data);
            this.updateLayer(name, this.activeLayers[name]);
        }

    };

    Handler.prototype.updateLayer = function (refName, data) {
        if(typeof refName !== 'undefined' && typeof this.layerRef[refName] !== 'undefined' && typeof this.layerRef[refName].ready !== 'undefined') {
            var that = this;
            this.layerRef[refName].ready.done(function() {
               that.layerRef[refName].instance.updateLayer(data);
            });
        } else {
            log.warn("Cannot update layer " + refName);
        }
    };
    
    
## Layers
    
    define(function (require) {
        'use strict';

        var $ = require('jquery');
        var log = require('logger');
        var ol = require('ol');

        var LayerPrototype = function() {
            this.data = this.pickle('data', {});
            this.persistent = this.pickle('persistent', false);
            this.layerOpacity = this.pickle('layerOpacity', 1);
            this.loopControls = this.pickle('loopControls', false);

            this.layersCreated = $.Deferred();
            this.currentLayer = 0;
            this.group = new ol.layer.Group({
                layers: []
            });
            this.layer = this.group;

            this.createLayers();
        };

        LayerPrototype.prototype.constructor = LayerPrototype;

        LayerPrototype.prototype.get = function(name) {
            return this[name];
        };

        LayerPrototype.prototype.createLayers = function() {
            var layers = [];
            layers.push(
                new ol.layer.Base({
                    source: new ol.source.TileDebug({
                    projection: 'EPSG:3857'
                })
            }));
            layer[0].getSource().setProperties({
                sourceInstance: undefined // not applicable without custom source (e.g. metar)
            });
            this.group.setLayers(new ol.Collection(layers));
            this.layersCreated.resolve();
        };

        LayerPrototype.prototype.updateLayer = function () {
            // do nothing by default (used for prototype/vector mostly)
        };

        LayerPrototype.prototype.showLayer = function (state) {
            state = this.persistent === true || typeof state === 'undefined' ? true : state;
            this.group.setVisible(state);
        };

        LayerPrototype.prototype.cycleLayer = function(step) { //-2, -1, 0, 1, 2, etc.
            step = step || 1;
            var layersColl = this.group.getLayers();
            layersColl.item(this.currentLayer).setOpacity(0);

            this.currentLayer = ((this.currentLayer + step)%10+10)%10; //((current+step)%10+10)%10
            var size = Object.keys(layersColl).length +1;
            var indicatorLen = ((this.currentLayer+1) / size) * 100 ; //determine length
            $("#counter_text span#x").html(this.currentLayer+1); //change text of frame loop txt
            $("#counter_indicator").css("width",indicatorLen+"%");
            layersColl.item(this.currentLayer).setOpacity(this.layerOpacity);
        };

        LayerPrototype.prototype.loop = function(action) {
            if(typeof this.loopInterval !== 'undefined') {
                clearInterval(this.loopInterval);
            }
            if(action === true || typeof action === 'undefined') {
                log.debug("Looping");
                var that = this;
                this.loopInterval = setInterval(function() {
                    that.cycleLayer();
                }, 500);
            }
        };


    // allows properties to be preserved (or pickled) if they are set in the child object that calls this parent prototype
    // otherwise it sets that property to the defaults in the parent
        LayerPrototype.prototype.pickle = function(name, value) {
            if( typeof this[name] !== 'undefined' ) {
                return this[name];
            } else {
                return value;
            }
        };

        return LayerPrototype;
    });


    // below are different child objects/classes that call the Layer prototypes 
    define(function(require) {
        'use strict';

        var ol = require('ol');
        var LayerPrototype = require('map/layer/prototype');

        var LayerXYZ = function(data) {
            this.data = this.pickle('data', data);

            LayerPrototype.call(this);
        };

        LayerXYZ.prototype = Object.create(LayerPrototype.prototype);
        LayerXYZ.prototype.constructor = LayerXYZ;

        return LayerXYZ;
    });

    define(function(require) {
        'use strict';

        var ol = require('ol');
        var log = require('logger');
        var LayerXYZ = require('map/layer/prototype/xyz');

        var LayerBackground = function() {
            this.persistent = true;

            LayerXYZ.call(this);
        };
        LayerBackground.prototype = Object.create(LayerXYZ.prototype);
        LayerBackground.prototype.constructor = LayerBackground;

        LayerBackground.prototype.createLayers = function() {
            var layers = [
                new ol.layer.Tile({
                    source: new ol.source.XYZ({
                        url: 'http://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}?appid=45ad401c-fa23-4a10-8b2f-a7ad29a3e2a0',
                    })
                })
            ];
            this.group.setLayers(new ol.Collection(layers));
            this.layersCreated.resolve();
        };

        return LayerBackground;
    });

    define(function(require) {
        'use strict';

        var ol = require('ol');
        var LayerXYZ = require('map/layer/prototype/xyz');
        var SourceReflective = require('map/source/tile.prototype/xyz/reflective');

        var ReflectiveLayer = function(data) {
            this.loopControls = true;
            this.data = data;
            this.layerOpacity = .5;

            LayerXYZ.call(this);
        };

        ReflectiveLayer.prototype = Object.create(LayerXYZ.prototype);
        ReflectiveLayer.prototype.constructor = ReflectiveLayer;

        ReflectiveLayer.prototype.createLayers = function() {
            var layers = [];

            for(var x=0; x<10; x++) {
                var sourceReflective = new SourceReflective(x);
                sourceReflective.source.setProperties({
                    instance: sourceReflective
                });
                layers.push(
                    new ol.layer.Tile({
                        source: sourceReflective.source,
                        opacity: 0
                }));
            };
            layers.reverse();
            layers[0].setOpacity(this.layerOpacity);
            $("#counter_text span#y").html(layers.length); //update the frame loop txt
            this.group.setLayers(new ol.Collection(layers));
            this.layersCreated.resolve();
        };

        return ReflectiveLayer;
    });


## Sources

    // This is a custom child of OpenLayers Sources, allowing use to add more properties and behaviors to our XYZ sources.

    define(function (require) {
        'use strict';

        var ol = require('ol');
        var SourcePrototype = require('map/source/tile.prototype');

        function SourceXYZ (iterator) {
            this.iterator = this.pickle('iterator', iterator);
            this.url = this.pickle('url', 'http://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}?appid=45ad401c-fa23-4a10-8b2f-a7ad29a3e2a0');

            SourcePrototype.call(this);
        }

        SourceXYZ.prototype = Object.create(SourcePrototype.prototype);
        SourceXYZ.prototype.constructor = SourceXYZ;

        SourceXYZ.prototype.createSource = function() {
            return new ol.source.XYZ({
                url: this.url,
                projection: this.projection
            });
        };

        return SourceXYZ;
    });

    // This is a specific child of the XYZ source prototype, to be used by our reflective radar layer

    define(function(require) {
        'use strict';

        var ol = require('ol');
        var SourceXYZPrototype = require('map/source/tile.prototype/xyz');

        var SourceReflective = function(iterator) {
            this.projection = 'EPSG:900913';
            this.iterator = iterator;
            this.url = 'http://ridgewms.srh.noaa.gov/tc/tc.py/1.0.0/N0Q_'+this.iterator+'/{z}/{x}/{y}';

            SourceXYZPrototype.call(this);
        };
        SourceReflective.prototype = Object.create(SourceXYZPrototype.prototype);
        SourceReflective.prototype.constructor = SourceReflective;

        return SourceReflective;
    })
    
    /* Specific Map instance */
    var thumbnailMap = new Map({
        target: 'baseReflective',
        controls: {attribution: false, zoom: false},
        playback: false
    });

    $(window).on('sector:set', function(evt){
        createRadarShortcut(evt);
    });

    function createRadarShortcut(evt) {
        $("#baseReflective").empty();

        var sectorData = session.getSectorData();

        thumbnailMap.setTarget();
        thumbnailMap.isReady().done(function() {
            thumbnailMap.setExtent(sectorData).done(function() {
                log.info("Updating map sector layer for new sector");
                thumbnailMap.updateMap(evt, {layer: "radar"});
            });
        });
    }