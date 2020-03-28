/*!
 * tw2overflow v2.0.0-dev
 * Tue, 18 Feb 2020 13:42:16 GMT
 * Developed by Relaxeaza <twoverflow@outlook.com>
 *
 * This work is free. You can redistribute it and/or modify it under the
 * terms of the Do What The Fuck You Want To Public License, Version 2,
 * as published by Sam Hocevar. See the LICENCE file for more details.
 */

;(function (window, undefined) {

const $rootScope = injector.get('$rootScope')
const transferredSharedDataService = injector.get('transferredSharedDataService')
const modelDataService = injector.get('modelDataService')
const socketService = injector.get('socketService')
const routeProvider = injector.get('routeProvider')
const eventTypeProvider = injector.get('eventTypeProvider')
const windowDisplayService = injector.get('windowDisplayService')
const windowManagerService = injector.get('windowManagerService')
const angularHotkeys = injector.get('hotkeys')
const armyService = injector.get('armyService')
const villageService = injector.get('villageService')
const mapService = injector.get('mapService')
const $filter = injector.get('$filter')
const $timeout = injector.get('$timeout')
const storageService = injector.get('storageService')
const reportService = injector.get('reportService')
const noop = function () {}

define('two/EventScope', [
    'queues/EventQueue'
], function (eventQueue) {
    const EventScope = function (windowId, onDestroy) {
        if (typeof windowId === 'undefined') {
            throw new Error('EventScope: no windowId')
        }

        this.windowId = windowId
        this.onDestroy = onDestroy || noop
        this.listeners = []

        const unregister = $rootScope.$on(eventTypeProvider.WINDOW_CLOSED, (event, templateName) => {
            if (templateName === '!' + this.windowId) {
                this.destroy()
                unregister()
            }
        })
    }

    EventScope.prototype.register = function (id, handler, _root) {
        if (_root) {
            this.listeners.push($rootScope.$on(id, handler))
        } else {
            eventQueue.register(id, handler)

            this.listeners.push(function () {
                eventQueue.unregister(id, handler)
            })
        }
    }

    EventScope.prototype.destroy = function () {
        this.listeners.forEach((unregister) => {
            unregister()
            this.onDestroy()
        })
    }

    return EventScope
})

define('two/utils', [
    'helper/time',
    'helper/math'
], function (
    $timeHelper,
    $math
) {
    let utils = {}

    /**
     * Gera um n첬mero aleat처rio aproximado da base.
     *
     * @param {Number} base - N첬mero base para o calculo.
     */
    utils.randomSeconds = function (base) {
        if (!base) {
            return 0
        }

        base = parseInt(base, 10)

        const max = base + (base / 2)
        const min = base - (base / 2)

        return Math.round(Math.random() * (max - min) + min)
    }

    /**
     * Converte uma string com um tempo em segundos.
     *
     * @param {String} time - Tempo que ser찼 convertido (hh:mm:ss)
     */
    utils.time2seconds = function (time) {
        time = time.split(':')
        time[0] = parseInt(time[0], 10) * 60 * 60
        time[1] = parseInt(time[1], 10) * 60
        time[2] = parseInt(time[2], 10)

        return time.reduce(function (a, b) {
            return a + b
        })
    }

    /**
     * Emite notifica챌찾o nativa do jogo.
     *
     * @param {String} type - success || error
     * @param {String} message - Texto a ser exibido
     */
    utils.emitNotif = function (type, message) {
        const eventType = type === 'success'
            ? eventTypeProvider.MESSAGE_SUCCESS
            : eventTypeProvider.MESSAGE_ERROR

        $rootScope.$broadcast(eventType, {
            message: message
        })
    }


    /**
     * Gera uma string com nome e coordenadas da aldeia
     *
     * @param {Object} village - Dados da aldeia
     * @return {String}
     */
    utils.genVillageLabel = function (village) {
        return village.name + ' (' + village.x + '|' + village.y + ')'
    }

    /**
     * Verifica se uma coordenada 챕 v찼lida.
     * 00|00
     * 000|00
     * 000|000
     * 00|000
     *
     * @param {String} xy - Coordenadas
     * @return {Boolean}
     */
    utils.isValidCoords = function (xy) {
        return /\s*\d{2,3}\|\d{2,3}\s*/.test(xy)
    }

    /**
     * Valida챌찾o de horario e data de envio. Exmplo: 23:59:00:999 30/12/2016
     *
     * @param  {String}  dateTime
     * @return {Boolean}
     */
    utils.isValidDateTime = function (dateTime) {
        return /^\s*([01][0-9]|2[0-3]):[0-5]\d:[0-5]\d(:\d{1,3})? (0[1-9]|[12][0-9]|3[0-1])\/(0[1-9]|1[0-2])\/\d{4}\s*$/.test(dateTime)
    }

    /**
     * Inverte a posi챌찾o do dia com o m챗s.
     */
    utils.fixDate = function (dateTime) {
        const dateAndTime = dateTime.split(' ')
        const time = dateAndTime[0]
        const date = dateAndTime[1].split('/')

        return time + ' ' + date[1] + '/' + date[0] + '/' + date[2]
    }

    /**
     * Gera um id unico
     *
     * @return {String}
     */
    utils.guid = function () {
        return Math.floor((Math.random()) * 0x1000000).toString(16)
    }

    /**
     * Verifica se um elemento 챕 pertencente a outro elemento.
     *
     * @param  {Element} elem - Elemento referencia
     * @param  {String} selector - Selector CSS do elemento no qual ser찼
     *   ser찼 verificado se tem rela챌찾o com o elemento indicado.
     * @return {Boolean}
     */
    utils.matchesElem = function (elem, selector) {
        if ($(elem).parents(selector).length) {
            return true
        }

        return false
    }

    /**
     * Obtem o timestamp de uma data em string.
     * Formato da data: m챗s/dia/ano
     * Exmplo de entrada: 23:59:59:999 12/30/2017
     *
     * @param  {String} dateString - Data em formato de string.
     * @return {Number} Timestamp (milisegundos)
     */
    utils.getTimeFromString = function (dateString, offset) {
        const dateSplit = dateString.trim().split(' ')
        const time = dateSplit[0].split(':')
        const date = dateSplit[1].split('/')

        const hour = time[0]
        const min = time[1]
        const sec = time[2]
        const ms = time[3] || null

        const month = parseInt(date[0], 10) - 1
        const day = date[1]
        const year = date[2]

        const _date = new Date(year, month, day, hour, min, sec, ms)

        return _date.getTime() + (offset || 0)
    }

    /**
     * Formata milisegundos em hora/data
     *
     * @return {String} Data e hora formatada
     */
    utils.formatDate = function (ms, format) {
        return $filter('readableDateFilter')(
            ms,
            null,
            $rootScope.GAME_TIMEZONE,
            $rootScope.GAME_TIME_OFFSET,
            format || 'HH:mm:ss dd/MM/yyyy'
        )
    }

    /**
     * Obtem a diferen챌a entre o timezone local e do servidor.
     *
     * @type {Number}
     */
    utils.getTimeOffset = function () {
        const localDate = $timeHelper.gameDate()
        const localOffset = localDate.getTimezoneOffset() * 1000 * 60
        const serverOffset = $rootScope.GAME_TIME_OFFSET

        return localOffset + serverOffset
    }

    utils.xhrGet = function (url, _callback, _dataType) {
        if (!url) {
            return false
        }

        _dataType = _dataType || 'text'
        _callback = _callback || noop

        let xhr = new XMLHttpRequest()
        xhr.open('GET', url, true)
        xhr.responseType = _dataType
        xhr.addEventListener('load', function () {
            _callback(xhr.response)
        }, false)

        xhr.send()
    }

    utils.obj2selectOptions = function (obj, _includeIcon) {
        let list = []

        for (let i in obj) {
            let item = {
                name: obj[i].name,
                value: obj[i].id
            }

            if (_includeIcon) {
                item.leftIcon = obj[i].icon
            }

            list.push(item)
        }

        return list
    }

    /**
     * @param {Object} origin - Objeto da aldeia origem.
     * @param {Object} target - Objeto da aldeia alvo.
     * @param {Object} units - Exercito usado no ataque como refer챗ncia
     * para calcular o tempo.
     * @param {String} type - Tipo de comando (attack,support,relocate)
     * @param {Object} officers - Oficiais usados no comando (usados para efeitos)
     *
     * @return {Number} Tempo de viagem
     */
    utils.getTravelTime = function (origin, target, units, type, officers) {
        let useEffects = false
        const targetIsBarbarian = target.character_id === null
        const targetIsSameTribe = target.character_id && target.tribe_id &&
                target.tribe_id === modelDataService.getSelectedCharacter().getTribeId()

        if (type === 'attack') {
            if ('supporter' in officers) {
                delete officers.supporter
            }

            if (targetIsBarbarian) {
                useEffects = true
            }
        } else if (type === 'support') {
            if (targetIsSameTribe) {
                useEffects = true
            }

            if ('supporter' in officers) {
                useEffects = true
            }
        }

        const army = {
            units: units,
            officers: angular.copy(officers)
        }

        const travelTime = armyService.calculateTravelTime(army, {
            barbarian: targetIsBarbarian,
            ownTribe: targetIsSameTribe,
            officers: officers,
            effects: useEffects
        }, type)

        const distance = $math.actualDistance(origin, target)

        const totalTravelTime = armyService.getTravelTimeForDistance(
            army,
            travelTime,
            distance,
            type
        )

        return totalTravelTime * 1000
    }

    return utils
})

define('two/ready', [
    'conf/gameStates'
], function (
    GAME_STATES
) {
    const ready = function (callback, which) {
        which = which || ['map']

        const readyStep = function (item) {
            which = which.filter(function (_item) {
                return _item !== item
            })

            if (!which.length) {
                callback()
            }
        }

        const handlers = {
            'map': function () {
                const mapScope = transferredSharedDataService.getSharedData('MapController')

                if (mapScope.isInitialized) {
                    return readyStep('map')
                }

                $rootScope.$on(eventTypeProvider.MAP_INITIALIZED, function () {
                    readyStep('map')
                })
            },
            'tribe_relations': function () {
                const $player = modelDataService.getSelectedCharacter()

                if ($player) {
                    const $tribeRelations = $player.getTribeRelations()

                    if (!$player.getTribeId() || $tribeRelations) {
                        return readyStep('tribe_relations')
                    }
                }

                const unbind = $rootScope.$on(eventTypeProvider.TRIBE_RELATION_LIST, function () {
                    unbind()
                    readyStep('tribe_relations')
                })
            },
            'initial_village': function () {
                const $gameState = modelDataService.getGameState()

                if ($gameState.getGameState(GAME_STATES.INITIAL_VILLAGE_READY)) {
                    return readyStep('initial_village')
                }

                $rootScope.$on(eventTypeProvider.GAME_STATE_INITIAL_VILLAGE_READY, function () {
                    readyStep('initial_village')
                })
            },
            'all_villages_ready': function () {
                const $gameState = modelDataService.getGameState()

                if ($gameState.getGameState(GAME_STATES.ALL_VILLAGES_READY)) {
                    return readyStep('all_villages_ready')
                }

                $rootScope.$on(eventTypeProvider.GAME_STATE_ALL_VILLAGES_READY, function () {
                    readyStep('all_villages_ready')
                })
            }
        }

        const mapScope = transferredSharedDataService.getSharedData('MapController')

        if (!mapScope) {
            return setTimeout(function () {
                ready(callback, which)
            }, 100)
        }

        which.forEach(function (readyItem) {
            handlers[readyItem]()
        })
    }

    return ready
})

require([
    'two/ready',
    'Lockr'
], function (
    ready,
    Lockr
) {
    ready(function () {
        let $player = modelDataService.getSelectedCharacter()

        // Lockr settings
        Lockr.prefix = $player.getId() + '_twOverflow_' + $player.getWorldId() + '-'
    })
})

define('two/language', ['helper/i18n'], function (i18n) {
    let initialized = false
    let moduleLanguages = {}
    let selectedLanguage
    const defaultLanguage = 'en_us'
    const sharedLanguages = {
        'en_dk': 'en_us',
        'pt_pt': 'pt_br'
    }

    const injectLanguage = function (moduleId) {
        if (!initialized) {
            throw new Error('Language module not initialized.')
            return false
        }

        if (!(moduleId in moduleLanguages)) {
            throw new Error('Language data for ' + moduleId + ' does not exists.')
            return false
        }

        const moduleLanguage = moduleLanguages[moduleId][selectedLanguage] || moduleLanguages[moduleId][defaultLanguage]

        i18n.setJSON(moduleLanguage)
    }

    const selectLanguage = function (language) {
        selectedLanguage = language

        if (selectedLanguage in sharedLanguages) {
            selectedLanguage = sharedLanguages[selectedLanguage]
        }

        return selectedLanguage
    }

    let twoLanguage = {}

    twoLanguage.add = function (moduleId, languages) {
        if (!initialized) {
            twoLanguage.init()
        }

        if (moduleId in moduleLanguages) {
            return false
        }

        moduleLanguages[moduleId] = languages

        injectLanguage(moduleId)
    }

    twoLanguage.init = function () {
        if (initialized) {
            return false
        }

        initialized = true
        selectedLanguage = selectLanguage($rootScope.loc.ale)

        // trigger eventTypeProvider.LANGUAGE_SELECTED_CHANGED you dumb fucks
        $rootScope.$watch('loc.ale', function (newValue, oldValue) {
            if (newValue === oldValue) {
                return false
            }

            selectLanguage($rootScope.loc.ale)

            for (let moduleId in moduleLanguages) {
                injectLanguage(moduleId)
            }
        })
    }

    return twoLanguage
})

define('two/Settings', [
    'Lockr'
], function (
    Lockr
) {
    const hasOwn = Object.prototype.hasOwnProperty

    const generateDiff = function (before, after) {
        let changes = {}

        for (let id in before) {
            if (hasOwn.call(after, id)) {
                if (!angular.equals(before[id], after[id])) {
                    changes[id] = after[id]
                }
            } else {
                changes[id] = before[id]
            }
        }

        return angular.equals({}, changes) ? false : changes
    }

    const generateDefaults = function (map) {
        let defaults = {}

        for (let key in map) {
            defaults[key] = map[key].default
        }

        return defaults
    }

    const disabledOption = function () {
        return {
            name: $filter('i18n')('disabled', $rootScope.loc.ale, 'common'),
            value: false
        }
    }

    const getUpdates = function (map, changes) {
        let updates = {}

        for (let id in changes) {
            (map[id].updates || []).forEach(function (updateItem) {
                updates[updateItem] = true
            })
        }

        if (angular.equals(updates, {})) {
            return false
        }

        return updates
    }

    let Settings = function (configs) {
        this.settingsMap = configs.settingsMap
        this.storageKey = configs.storageKey
        this.defaults = generateDefaults(this.settingsMap)
        this.settings = angular.merge({}, this.defaults, Lockr.get(this.storageKey, {}))
        this.events = {
            settingsChange: noop
        }
        this.injected = false
    }

    Settings.prototype.get = function (id) {
        return this.settings[id]
    }

    Settings.prototype.getAll = function () {
        return this.settings
    }

    Settings.prototype.getDefault = function (id) {
        return hasOwn.call(this.defaults, id) ? this.defaults[id] : undefined
    }

    Settings.prototype.store = function () {
        Lockr.set(this.storageKey, this.settings)
    }

    Settings.prototype.set = function (id, value, opt) {
        if (!hasOwn.call(this.settingsMap, id)) {
            return false
        }

        const before = angular.copy(this.settings)
        this.settings[id] = value
        const after = angular.copy(this.settings)
        const changes = generateDiff(before, after)

        if (!changes) {
            return false
        }

        const updates = getUpdates(this.settingsMap, changes)

        this.store()
        this.updateScope()
        this.events.settingsChange(changes, updates, opt || {})

        return true
    }

    Settings.prototype.setAll = function (values, opt) {
        const before = angular.copy(this.settings)

        for (let id in values) {
            if (hasOwn.call(this.settingsMap, id)) {
                this.settings[id] = values[id]
            }
        }

        const after = angular.copy(this.settings)
        const changes = generateDiff(before, after)

        if (!changes) {
            return false
        }

        const updates = getUpdates(this.settingsMap, changes)

        this.store()
        this.updateScope()
        this.events.settingsChange(changes, updates, opt || {})

        return true
    }

    Settings.prototype.reset = function (id, opt) {
        this.set(id, this.defaults[id], opt)

        return true
    }

    Settings.prototype.resetAll = function (opt) {
        this.setAll(angular.copy(this.defaults), opt)

        return true
    }

    Settings.prototype.each = function (callback) {
        for (let id in this.settings) {
            let map = this.settingsMap[id]

            if (map.inputType === 'checkbox') {
                callback.call(this, id, !!this.settings[id], map)
            } else {
                callback.call(this, id, this.settings[id], map)
            }
        }
    }

    Settings.prototype.onChange = function (callback) {
        if (typeof callback === 'function') {
            this.events.settingsChange = callback
        }
    }

    Settings.prototype.injectScope = function ($scope, opt) {
        this.injected = {
            $scope: $scope,
            opt: opt
        }

        $scope.settings = this.encode(opt)

        angular.forEach(this.settingsMap, function (map, id) {
            if (map.inputType === 'select') {
                $scope.$watch(function () {
                    return $scope.settings[id]
                }, function (value) {
                    if (map.multiSelect) {
                        if (!value.length) {
                            $scope.settings[id] = [disabledOption()]
                        }
                    } else if (!value) {
                        $scope.settings[id] = disabledOption()
                    }
                }, true)
            }
        })
    }

    Settings.prototype.updateScope = function () {
        if (!this.injected) {
            return false
        }

        this.injected.$scope.settings = this.encode(this.injected.opt)
    }

    Settings.prototype.encode = function (opt) {
        let encoded = {}
        const presets = modelDataService.getPresetList().getPresets()
        const groups = modelDataService.getGroupList().getGroups()

        opt = opt || {}

        this.each(function (id, value, map) {
            if (map.inputType === 'select') {
                if (!value && map.disabledOption) {
                    encoded[id] = map.multiSelect ? [disabledOption()] : disabledOption()
                    return
                }

                switch (map.type) {
                case 'presets':
                    if (map.multiSelect) {
                        let multiValues = []

                        value.forEach(function (presetId) {
                            if (!presets[presetId]) {
                                return
                            }

                            multiValues.push({
                                name: presets[presetId].name,
                                value: presetId
                            })
                        })

                        encoded[id] = multiValues.length ? multiValues : [disabledOption()]
                    } else {
                        if (!presets[value] && map.disabledOption) {
                            encoded[id] = disabledOption()
                            return
                        }

                        encoded[id] = {
                            name: presets[value].name,
                            value: value
                        }
                    }

                    break
                case 'groups':
                    if (map.multiSelect) {
                        let multiValues = []

                        value.forEach(function (groupId) {
                            if (!groups[groupId]) {
                                return
                            }

                            multiValues.push({
                                name: groups[groupId].name,
                                value: groupId,
                                leftIcon: groups[groupId].icon
                            })
                        })

                        encoded[id] = multiValues.length ? multiValues : [disabledOption()]
                    } else {
                        if (!groups[value] && map.disabledOption) {
                            encoded[id] = disabledOption()
                            return
                        }

                        encoded[id] = {
                            name: groups[value].name,
                            value: value
                        }
                    }

                    break
                default:
                    encoded[id] = {
                        name: opt.textObject ? $filter('i18n')(value, $rootScope.loc.ale, opt.textObject) : value,
                        value: value
                    }

                    if (opt.multiSelect) {
                        encoded[id] = [encoded[id]]
                    }

                    break
                }
            } else {
                encoded[id] = value
            }
        })

        return encoded
    }

    Settings.prototype.decode = function (encoded) {
        let decoded = {}

        for (let id in encoded) {
            let map = this.settingsMap[id]

            if (map.inputType === 'select') {
                if (map.multiSelect) {
                    if (encoded[id].length === 1 && encoded[id][0].value === false) {
                        decoded[id] = []
                    } else {
                        let multiValues = []

                        encoded[id].forEach(function (item) {
                            multiValues.push(item.value)
                        })

                        decoded[id] = multiValues
                    }
                } else {
                    decoded[id] = encoded[id].value
                }
            } else {
                decoded[id] = encoded[id]
            }
        }

        return decoded
    }

    Settings.encodeList = function (list, opt) {
        let encoded = []

        opt = opt || {}

        if (opt.disabled) {
            encoded.push(disabledOption())
        }

        switch (opt.type) {
        case 'keys':
            for (let prop in list) {
                let value = list[prop]

                encoded.push({
                    name: prop,
                    value: prop
                })
            }

            break
        case 'groups':
            for (let prop in list) {
                let value = list[prop]

                encoded.push({
                    name: value.name,
                    value: value.id,
                    leftIcon: value.icon
                })
            }

            break
        case 'presets':
            for (let prop in list) {
                let value = list[prop]

                encoded.push({
                    name: value.name,
                    value: value.id
                })
            }

            break
        case 'values':
        default:
            for (let prop in list) {
                let value = list[prop]

                encoded.push({
                    name: opt.textObject ? $filter('i18n')(value, $rootScope.loc.ale, opt.textObject) : value,
                    value: value
                })
            }
        }

        return encoded
    }

    Settings.disabledOption = disabledOption

    return Settings
})

define('two/mapData', [
    'conf/conf',
    'states/MapState'
], function (
    conf,
    mapState
) {
    let villages = []
    let width = 306
    let height = 306
    let grid = []
    let loadingQueue = {}

    const init = function () {
        setupGrid()
    }

    const setupGrid = function () {
        const xChunks = Math.ceil(conf.MAP_SIZE / width)
        const yChunks = Math.ceil(conf.MAP_SIZE / height)

        for (let gridX = 0; gridX < xChunks; gridX++) {
            grid.push([])

            let chunkX = width * gridX
            let chunkWidth = width.bound(0, chunkX + width).bound(0, conf.MAP_SIZE - chunkX)
            chunkX = chunkX.bound(0, conf.MAP_SIZE)

            for (let gridY = 0; gridY < yChunks; gridY++) {
                let chunkY = height * gridY
                let chunkHeight = height.bound(0, chunkY + height).bound(0, conf.MAP_SIZE - chunkY)
                chunkY = chunkY.bound(0, conf.MAP_SIZE)

                grid[gridX].push({
                    x: chunkX,
                    y: chunkY,
                    width: chunkWidth,
                    height: chunkHeight,
                    loaded: false,
                    loading: false
                })
            }
        }
    }

    const getVisibleGridCells = function (origin) {
        let cells = []

        for (let gridX = 0; gridX < grid.length; gridX++) {
            for (let gridY = 0; gridY < grid[gridX].length; gridY++) {
                let cell = grid[gridX][gridY]

                if (
                    Math.abs(origin.x) + width <= cell.x ||
                    Math.abs(origin.y) + height <= cell.y ||
                    Math.abs(origin.x) >= cell.x + cell.width ||
                    Math.abs(origin.y) >= cell.y + cell.height
                ) {
                    continue
                }

                cells.push(cell)
            }
        }

        return cells
    }

    const genLoadingId = function (cells) {
        let id = []

        cells.forEach(function (cell) {
            id.push(cell.x.toString() + cell.y.toString())
        })

        return id.join('')
    }

    let mapData = {}

    mapData.load = function (origin, callback, _error) {
        const cells = getVisibleGridCells(origin)
        let loadId = genLoadingId(cells)
        let requests = []
        const alreadyLoading = cells.every(function (cell) {
            return cell.loading
        })

        loadingQueue[loadId] = loadingQueue[loadId] || []
        loadingQueue[loadId].push(callback)

        if (alreadyLoading) {
            return
        }

        cells.forEach(function (cell) {
            if (cell.loaded || cell.loading) {
                return
            }

            cell.loading = true

            let promise = new Promise(function (resolve, reject) {
                socketService.emit(routeProvider.MAP_GET_MINIMAP_VILLAGES, cell, function (data) {
                    cell.loaded = true
                    cell.loading = false

                    if (data.message) {
                        return reject(data.message)
                    }

                    if (data.villages.length) {
                        villages = villages.concat(data.villages)
                    }

                    resolve(cell)
                })
            })

            requests.push(promise)
        })

        if (!requests.length) {
            return callback(villages)
        }

        Promise.all(requests)
        .then(function (cells) {
            loadId = genLoadingId(cells)

            loadingQueue[loadId].forEach(function (handler) {
                handler(villages)
            })

            delete loadingQueue[loadId]
        })
        .catch(function (error) {
            console.error(error.message)
        })
    }

    mapData.getVillages = function () {
        return villages
    }

    init()

    return mapData
})

require([
    'two/language',
    'two/ready'
], function (
    twoLanguage,
    ready
) {
    ready(function () {
        twoLanguage.add('core', {"cs_cz":{"common":{"start":"Spustit","started":"Started","pause":"Pozastavit","paused":"Pozastaveno","stop":"Stop","stopped":"Stopped","status":"Stav","none":"탐찼dn첵","info":"Informace","settings":"Nastaven챠","others":"Ostatn챠","village":"Village","villages":"Vesnice","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filtry","add":"P힂idat","waiting":"Vy훾k찼v찼","attack":"횣tok","support":"Podpora","relocate":"P힂esun","activate":"Activate","deactivate":"Zru큄it","units":"Jednotky","officers":"D킁stojn챠ci","origin":"P킁vod","target":["Target","Targets"],"save":"Ulo탑it","logs":"Logs","headquarter":"Velitelstv챠","barracks":"Kas찼rny","tavern":"Hospoda","hospital":"Nemocnice","preceptory":"Komturie","chapel":"Kaple","church":"Kostel","academy":"Akademie","rally_point":"Shroma탑di큄t휎","statue":"Socha","market":"Tr탑nice","timber_camp":"D힂evorubec","clay_pit":"J챠lovi큄t휎","iron_mine":"D킁l na 탑elezo","farm":"Farma","warehouse":"Sklad","wall":"Ze휁","spear":"Kopin챠k","sword":"힋erm챠힂","axe":"Sekern챠k","archer":"Lukost힂elec","light_cavalry":"Lehk찼 j챠zda","mounted_archer":"J챠zdn챠 lu훾i큄tn챠k","heavy_cavalry":"T휎탑k찼 j챠zka","ram":"Beranidlo","catapult":"Katapult","doppelsoldner":"Bersekr","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"da_dk":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"de_de":{"common":{"start":"Start","started":"Gestartet","pause":"Pause","paused":"Pausiert","stop":"Anhalten","stopped":"Angehalten","status":"Status","none":"Keine","info":"Informationen","settings":"Einstellungen","others":"Andere","village":"Dorf","villages":"D철rfer","building":"Geb채ude","buildings":"Geb채ude","level":"Stufe","registers":"Protokoll","filters":"Filter","add":"Hinzuf체gen","waiting":"Ausstehend","attack":"Angriff","support":"Unterst체tzung","relocate":"Umsiedlung","activate":"Aktivieren","deactivate":"Deaktivieren","units":"Einheiten","officers":"Offiziere","origin":"Herkunft","target":["Ziel","Ziele"],"save":"Speichern","logs":"Protokoll","headquarter":"Hauptgeb채ude","barracks":"Kaserne","tavern":"Taverne","hospital":"Lazarett","preceptory":"Ordenshalle","chapel":"Kapelle","church":"Kirche","academy":"Akademie","rally_point":"Sammelplatz","statue":"Statue","market":"Markt","timber_camp":"Holzf채ller","clay_pit":"Lehmgrube","iron_mine":"Eisenmine","farm":"Bauernhof","warehouse":"Speicher","wall":"Wall","spear":"Speerk채mpfer","sword":"Schwertk채mpfer","axe":"Axtk채mpfer","archer":"Bogensch체tzen","light_cavalry":"Leichte Kavallerie","mounted_archer":"Berittener Bogensch체tze","heavy_cavalry":"Schwere Kavallerie","ram":"Rammbock","catapult":"Katapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Adelsgeschlecht","knight":"Paladin","no-results":"Keine Ergebnisse...","selected":"Ausgew채hlt","now":"Jetzt","costs":"Kosten","duration":"Dauer","points":"Punkte","player":"Spieler","players":"Spieler","next_features":"Kommende Features","misc":"Verschiedenes","colors":"Farben","reset":"Zur체cksetzen","here":"hier","disabled":"�� Deaktiviert ��","cancel":"Abbrechen","actions":"Aktionen","remove":"Entfernen","started_at":"Gestartet um","arrive":"Ankommen","settings_saved":"Einstellungen gespeichert"}},"el_gr":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"en_us":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"es_es":{"common":{"start":"Comenzar","started":"Started","pause":"Pausa","paused":"Pausado","stop":"Stop","stopped":"Stopped","status":"Estatus","none":"Ninguno","info":"Informaci처n","settings":"Controles","others":"Otros","village":"Village","villages":"Villas","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filtros","add":"Agregar","waiting":"Esperando","attack":"Ataque","support":"Soporte","relocate":"Transferir","activate":"Activate","deactivate":"Inhabilitado","units":"Unidades","officers":"Oficiales","origin":"Origen","target":["Target","Targets"],"save":"Guardar","logs":"Logs","headquarter":"Sede general","barracks":"Cuarteles","tavern":"Taberna","hospital":"Hospital","preceptory":"Sala de Ordenes","chapel":"Capilla","church":"Iglesia","academy":"Academia","rally_point":"Punto de reuni처n","statue":"Estatua","market":"Mercado","timber_camp":"Campamento de madera","clay_pit":"Pozo de Arcilla","iron_mine":"Mina de Hierro","farm":"Granja","warehouse":"Dep처sito","wall":"Pared","spear":"Lancero","sword":"Espadach챠n","axe":"Luchador de Hacha","archer":"Arquero","light_cavalry":"Caballer챠a ligera","mounted_archer":"Arquer챠a montada","heavy_cavalry":"Caballer챠a Pesada","ram":"Martillo","catapult":"Catapulta","doppelsoldner":"Fren챕tico","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"fi_fi":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"fr_fr":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"hu_hu":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"it_it":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"nb_no":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"nl_nl":{"common":{"start":"Start","started":"Gestart","pause":"Pauze","paused":"Gepauzeerd","stop":"Stop","stopped":"Gestopt","status":"Status","none":"Geen","info":"Informatie","settings":"Instellingen","others":"Anderen","village":"Dorp","villages":"Dorpen","building":"Gebouw","buildings":"Gebouwen","level":"Niveau","registers":"Logboeken","filters":"Filters","add":"Toevoegen","waiting":"Wachten","attack":"Aanvallen","support":"Ondersteunen","relocate":"Verplaatsen","activate":"Activeren","deactivate":"Uitschakelen","units":"Eenheden","officers":"Officieren","origin":"Herkomst","target":["Doel","Doelen"],"save":"Opslaan","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"pl_pl":{"common":{"start":"Start","started":"Uruchomiony","pause":"Pauza","paused":"Wstrzymany","stop":"Zatrzymany","stopped":"Zatrzymany","status":"Status","none":"탈aden","info":"Informacje","settings":"Ustawienia","others":"Inne","village":"Wioska","villages":"Wioski","building":"Budynek","buildings":"Budynki","level":"Poziom","registers":"Logs","filters":"Filtry","add":"Dodaj","waiting":"Oczekuj훳ce","attack":"Atak","support":"Wsparcie","relocate":"Przeniesienie","activate":"Activate","deactivate":"Wy흢훳cz","units":"Jednostki","officers":"Oficerowie","origin":"탁r처d흢o","target":["Target","Targets"],"save":"Zapisz","logs":"Logi","headquarter":"Ratusz","barracks":"Koszary","tavern":"Tawerna","hospital":"Szpital","preceptory":"Komturia","chapel":"Kaplica","church":"Ko힄ci처흢","academy":"Akademia","rally_point":"Plac","statue":"Piedesta흢","market":"Rynek","timber_camp":"Tartak","clay_pit":"Kopalnia gliny","iron_mine":"Huta 탉elaza","farm":"Farma","warehouse":"Magazyn","wall":"Mur","spear":"Pikinier","sword":"Miecznik","axe":"Topornik","archer":"흟ucznik","light_cavalry":"Lekki kawalerzysta","mounted_archer":"흟ucznik konny","heavy_cavalry":"Ci휌탉ki kawalerzysta","ram":"Taran","catapult":"Katapulta","doppelsoldner":"Berserker","trebuchet":"Trebusz","snob":"Szlachcic","knight":"Rycerz","no-results":"Brak wynik처w...","selected":"Wybrana","now":"Teraz","costs":"Koszty","duration":"Czas trwania","points":"Punkty","player":"Gracz","players":"Gracze","next_features":"Nast휌pne funkcje","misc":"R처탉ne","colors":"Kolory","reset":"Reset","here":"here","disabled":"�� Wy흢훳czony ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Ustawienia zapisane"}},"pt_br":{"common":{"start":"Iniciar","started":"Iniciado","pause":"Pausar","paused":"Pausado","stop":"Parar","stopped":"Parado","status":"Status","none":"Nenhum","info":"Informa챌천es","settings":"Configura챌천es","others":"Outros","village":"Aldeia","villages":"Aldeias","building":"Edif챠cio","buildings":"Edif챠cios","level":"N챠vel","registers":"Registros","filters":"Filtros","add":"Adicionar","waiting":"Em espera","attack":"Ataque","support":"Apoio","relocate":"Transfer챗ncia","activate":"Ativar","deactivate":"Desativar","units":"Unidades","officers":"Oficiais","origin":"Origem","target":["Alvo","Alvos"],"save":"Salvar","logs":"Eventos","headquarter":"Edif챠cio Principal","barracks":"Quartel","tavern":"Taverna","hospital":"Hospital","preceptory":"Sal찾o das Ordens","chapel":"Capela","church":"Igreja","academy":"Academia","rally_point":"Ponto de Encontro","statue":"Est찼tua","market":"Mercado","timber_camp":"Bosque","clay_pit":"Po챌o de Argila","iron_mine":"Mina de Ferro","farm":"Fazenda","warehouse":"Armaz챕m","wall":"Muralha","spear":"Lanceiro","sword":"Espadachim","axe":"Viking","archer":"Arqueiro","light_cavalry":"Cavalaria Leve","mounted_archer":"Arqueiro Montado","heavy_cavalry":"Cavalaria Pesada","ram":"Ar챠ete","catapult":"Catapulta","doppelsoldner":"Berserker","trebuchet":"Trabuco","snob":"Nobre","knight":"Paladino","no-results":"Sem resultados...","selected":"Selecionado","now":"Agora","costs":"Custos","duration":"Dura챌찾o","points":"Pontos","player":"Jogador","players":"Jogadores","next_features":"Pr처ximas funcionalidades","misc":"Diversos","colors":"Cores","reset":"Resetar","here":"aqui","disabled":"�� Desativado ��","cancel":"Cancelar","actions":"A챌천es","remove":"Remover","started_at":"Iniciado em","arrive":"Chegada","settings_saved":"Configura챌천es salvas"}},"ro_ro":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"ru_ru":{"common":{"start":"�逵�逵��","started":"�逵極��筠戟","pause":"�逵�鈞逵","paused":"��龜棘��逵戟棘勻剋筠戟","stop":"鬼�棘極","stopped":"���逵戟棘勻剋筠戟","status":"鬼�逵���","none":"������勻�筠�","info":"�戟�棘�劇逵�龜�","settings":"�逵���棘橘克龜","others":"��棘�筠筠","village":"�筠�筠勻戟�","villages":"�筠�筠勻戟龜","building":"鬼��棘筠戟龜筠","buildings":"鬼��棘筠戟龜�","level":"叫�棘勻筠戟�","registers":"���戟逵剋 �棘閨��龜橘","filters":"圭龜剋����","add":"�棘閨逵勻龜��","waiting":"�菌龜畇逵戟龜筠","attack":"��逵克逵","support":"�棘畇畇筠�菌克逵","relocate":"�筠�筠劇筠�筠戟龜筠","activate":"�克�龜勻龜�棘勻逵��","deactivate":"��克剋��龜��","units":"�棘橘�克逵","officers":"�棘筠戟逵�逵剋�戟龜克龜","origin":"龜棘�克逵 棘�極�逵勻剋筠戟龜�","target":["龜棘�克逵 戟逵鈞戟逵�筠戟龜�","揆筠剋龜"],"save":"鬼棘��逵戟龜��","logs":"���戟逵剋 �棘閨��龜橘","headquarter":"�逵���逵","barracks":"�逵鈞逵�劇�","tavern":"龜逵勻筠�戟逵","hospital":"�棘�極龜�逵剋�","preceptory":"�逵剋 棘�畇筠戟棘勻","chapel":"槻逵�棘勻戟�","church":"揆筠�克棘勻�","academy":"�克逵畇筠劇龜�","rally_point":"�剋棘�逵畇�","statue":"鬼�逵���","market":"��戟棘克","timber_camp":"�筠�棘極龜剋克逵","clay_pit":"�剋龜戟�戟�橘 克逵��筠�","iron_mine":"�筠剋筠鈞戟�橘 ��畇戟龜克","farm":"圭筠�劇逵","warehouse":"鬼克剋逵畇","wall":"鬼�筠戟逵","spear":"�棘極筠橘�龜克","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"sk_sk":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"sv_se":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}},"tr_tr":{"common":{"start":"Start","started":"Started","pause":"Pause","paused":"Paused","stop":"Stop","stopped":"Stopped","status":"Status","none":"None","info":"Information","settings":"Settings","others":"Others","village":"Village","villages":"Villages","building":"Building","buildings":"Buildings","level":"Level","registers":"Logs","filters":"Filters","add":"Add","waiting":"Waiting","attack":"Attack","support":"Support","relocate":"Transfer","activate":"Activate","deactivate":"Disable","units":"Units","officers":"Officers","origin":"Origin","target":["Target","Targets"],"save":"Save","logs":"Logs","headquarter":"Headquarters","barracks":"Barracks","tavern":"Tavern","hospital":"Hospital","preceptory":"Hall of Orders","chapel":"Chapel","church":"Church","academy":"Academy","rally_point":"Rally Point","statue":"Statue","market":"Market","timber_camp":"Timber Camp","clay_pit":"Clay Pit","iron_mine":"Iron Mine","farm":"Farm","warehouse":"Warehouse","wall":"Wall","spear":"Spearman","sword":"Swordsman","axe":"Axe Fighter","archer":"Archer","light_cavalry":"Light Cavalry","mounted_archer":"Mounted Archer","heavy_cavalry":"Heavy Cavalry","ram":"Ram","catapult":"Catapult","doppelsoldner":"Berserker","trebuchet":"Trebuchet","snob":"Nobleman","knight":"Paladin","no-results":"No results...","selected":"Selected","now":"Now","costs":"Costs","duration":"Duration","points":"Points","player":"Player","players":"Players","next_features":"Next features","misc":"Miscellaneous","colors":"Colors","reset":"Reset","here":"here","disabled":"�� Disabled ��","cancel":"Cancel","actions":"Actions","remove":"Remove","started_at":"Started at","arrive":"Arrive","settings_saved":"Settings saved"}}})
    })
})

/**
 * https://github.com/tsironis/lockr
 */
;(function(root, factory) {
    define('Lockr', factory(root, {}))
}(this, function(root, Lockr) {
    'use strict'

    Lockr.prefix = ''

    Lockr._getPrefixedKey = function(key, options) {
        options = options || {}

        if (options.noPrefix) {
            return key
        } else {
            return this.prefix + key
        }

    }

    Lockr.set = function(key, value, options) {
        const query_key = this._getPrefixedKey(key, options)

        try {
            localStorage.setItem(query_key, JSON.stringify({
                data: value
            }))
        } catch (e) {}
    }

    Lockr.get = function(key, missing, options) {
        const query_key = this._getPrefixedKey(key, options)
        let value

        try {
            value = JSON.parse(localStorage.getItem(query_key))
        } catch (e) {
            if (localStorage[query_key]) {
                value = {
                    data: localStorage.getItem(query_key)
                }
            } else {
                value = null
            }
        }

        if (value === null) {
            return missing
        } else if (typeof value === 'object' && typeof value.data !== 'undefined') {
            return value.data
        } else {
            return missing
        }
    }

    return Lockr
}))

define('two/about', [], function () {
    let initialized = false

    let about = {}

    about.isInitialized = function () {
        return initialized
    }

    about.init = function () {
        initialized = true
    }

    return about
})

define('two/about/ui', [
    'two/ui'
], function (
    interfaceOverflow
) {
    let $scope

    const selectTab = function (tabType) {
        $scope.selectedTab = tabType
    }

    const init = function () {
        $opener = interfaceOverflow.addMenuButton('About', 100)

        $opener.addEventListener('click', function () {
            buildWindow()
        })

        interfaceOverflow.addTemplate('twoverflow_about_window', `<div id=\"two-about\" class=\"win-content\">
													<header class=\"win-head\">
														<h3>tw2overflow v2.0.0-dev</h3>
														<ul class=\"list-btn sprite\">
															<li>
																<a href=\"#\" class=\"btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
															</ul>
														</header>
														<div class=\"win-main\" scrollbar=\"\">
															<div class=\"box-paper footer\">
																<div class=\"scroll-wrap\">
																	<div class=\"logo\">
																		<img src=\"https://i.imgur.com/iNcVMvw.png\">
																		</div>
																		<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																			<tr>
																				<th colspan=\"2\">{{ 'contact' | i18n:loc.ale:'about' }}<tr>
																						<td>{{ 'email' | i18n:loc.ale:'about' }}<td>twoverflow@outlook.com</table>
																							<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																								<tr>
																									<th colspan=\"2\">{{ 'links' | i18n:loc.ale:'about' }}<tr>
																											<td>{{ 'source_code' | i18n:loc.ale:'about' }}<td>
																													<a href=\"https://gitlab.com/relaxeaza/twoverflow/\" target=\"_blank\">https://gitlab.com/relaxeaza/twoverflow/</a>
																													<tr>
																														<td>{{ 'issues_suggestions' | i18n:loc.ale:'about' }}<td>
																																<a href=\"https://gitlab.com/relaxeaza/twoverflow/issues\" target=\"_blank\">https://gitlab.com/relaxeaza/twoverflow/issues</a>
																																<tr>
																																	<td>{{ 'translations' | i18n:loc.ale:'about' }}<td>
																																			<a href=\"https://crowdin.com/project/twoverflow\" target=\"_blank\">https://crowdin.com/project/twoverflow</a>
																																		</table>
																																	</div>
																																</div>
																															</div>
																															<footer class=\"win-foot\">
																																<ul class=\"list-btn list-center\">
																																	<li>
																																		<a href=\"#\" class=\"btn-border btn-red\" ng-click=\"closeWindow()\">{{ 'cancel' | i18n:loc.ale:'common' }}</a>
																																	</ul>
																																</footer>
																															</div>`)
        interfaceOverflow.addStyle('#two-about{padding:42px 0 0px 0;position:relative;height:100%}#two-about .box-paper a{font-weight:bold;color:#3f2615;text-decoration:none}#two-about .box-paper a:hover{text-shadow:0 1px 0 #000;color:#fff}#two-about .logo{text-align:center;margin-bottom:8px}#two-about table td{padding:0 10px}#two-about table td:first-child{text-align:right;width:20%}')
    }

    const buildWindow = function () {
        $scope = $rootScope.$new()
        $scope.selectTab = selectTab

        windowManagerService.getModal('!twoverflow_about_window', $scope)
    }

    return init
})

require([
    'two/language',
    'two/ready',
    'two/about',
    'two/about/ui'
], function (
    twoLanguage,
    ready,
    about,
    aboutInterface
) {
    if (about.isInitialized()) {
        return false
    }

    ready(function () {
        twoLanguage.add('about', {"de_de":{"about":{"contact":"Contact","email":"E-mail","links":"Project links","source_code":"Source Code","issues_suggestions":"Issues/suggestions","translations":"Translations"}},"en_us":{"about":{"contact":"Contact","email":"E-mail","links":"Project links","source_code":"Source Code","issues_suggestions":"Issues/suggestions","translations":"Translations"}},"pl_pl":{"about":{"contact":"Contact","email":"E-mail","links":"Project links","source_code":"Source Code","issues_suggestions":"Issues/suggestions","translations":"Translations"}},"pt_br":{"about":{"contact":"Contato","email":"E-mail","links":"Links do projeto","source_code":"C처digo Fonte","issues_suggestions":"Id챕ias/Sugest천es","translations":"Tradu챌천es"}},"ru_ru":{"about":{"contact":"Contact","email":"E-mail","links":"Project links","source_code":"Source Code","issues_suggestions":"Issues/suggestions","translations":"Translations"}}})
        about.init()
        aboutInterface()
    }, ['map'])
})

define('two/attackView', [
    'two/ready',
    'two/utils',
    'two/attackView/types/columns',
    'two/attackView/types/commands',
    'two/attackView/types/filters',
    'two/attackView/unitSpeedOrder',
    'conf/unitTypes',
    'Lockr',
    'helper/math',
    'helper/mapconvert',
    'struct/MapData',
    'queues/EventQueue',
    'models/CommandModel'
], function (
    ready,
    utils,
    COLUMN_TYPES,
    COMMAND_TYPES,
    FILTER_TYPES,
    UNIT_SPEED_ORDER,
    UNIT_TYPES,
    Lockr,
    math,
    convert,
    mapData,
    eventQueue,
    CommandModel
) {
    let initialized = false
    let listeners = {}
    let overviewService = injector.get('overviewService')
    let globalInfoModel
    let commands = []
    let commandQueue = false
    let filters = {}
    let filterParams = {}
    let sorting = {
        reverse: false,
        column: COLUMN_TYPES.TIME_COMPLETED
    }
    const COMMAND_ORDER = [
        'ATTACK',
        'SUPPORT',
        'RELOCATE'
    ]
    const STORAGE_KEYS = {
        FILTERS: 'attack_view_filters'
    }
    const INCOMING_UNITS_FILTER = {}
    const COMMAND_TYPES_FILTER = {}

    const resetFilters = function () {
        filters = {}
        filters[FILTER_TYPES.COMMAND_TYPES] = {}
        filters[FILTER_TYPES.VILLAGE] = angular.copy(COMMAND_TYPES_FILTER)
        filters[FILTER_TYPES.INCOMING_UNITS] = angular.copy(INCOMING_UNITS_FILTER)
    }

    const formatFilters = function () {
        const toArray = [FILTER_TYPES.COMMAND_TYPES]
        const currentVillageId = modelDataService.getSelectedVillage().getId()
        let arrays = {}

        // format filters for backend
        for (let i = 0; i < toArray.length; i++) {
            for (let j in filters[toArray[i]]) {
                if (!arrays[toArray[i]]) {
                    arrays[toArray[i]] = []
                }

                if (filters[toArray[i]][j]) {
                    switch (toArray[i]) {
                    case FILTER_TYPES.COMMAND_TYPES:
                        if (j === COMMAND_TYPES.ATTACK) {
                            arrays[toArray[i]].push(COMMAND_TYPES.ATTACK)
                        } else if (j === COMMAND_TYPES.SUPPORT) {
                            arrays[toArray[i]].push(COMMAND_TYPES.SUPPORT)
                        } else if (j === COMMAND_TYPES.RELOCATE) {
                            arrays[toArray[i]].push(COMMAND_TYPES.RELOCATE)
                        }
                        break
                    }
                }
            }
        }

        filterParams = arrays
        filterParams.village = filters[FILTER_TYPES.VILLAGE] ? [currentVillageId] : []
    }

    /**
     * Command was sent.
     */
    const onCommandIncomming = function () {
        // we can never know if the command is currently visible (because of filters, sorting and stuff) -> reload
        attackView.loadCommands()
    }

    /**
     * Command was cancelled.
     *
     * @param {Object} event unused
     * @param {Object} data The backend-data
     */
    const onCommandCancelled = function (event, data) {
        eventQueue.trigger(eventTypeProvider.ATTACK_VIEW_COMMAND_CANCELLED, [data.id || data.command_id])
    }

    /**
     * Command ignored.
     *
     * @param {Object} event unused
     * @param {Object} data The backend-data
     */
    const onCommandIgnored = function (event, data) {
        for (let i = 0; i < commands.length; i++) {
            if (commands[i].command_id === data.command_id) {
                commands.splice(i, 1)
            }
        }

        eventQueue.trigger(eventTypeProvider.ATTACK_VIEW_COMMAND_IGNORED, [data.command_id])
    }

    /**
     * Village name changed.
     *
     * @param {Object} event unused
     * @param {Object} data The backend-data
     */
    const onVillageNameChanged = function (event, data) {
        for (let i = 0; i < commands.length; i++) {
            if (commands[i].target_village_id === data.village_id) {
                commands[i].target_village_name = data.name
                commands[i].targetVillage.name = data.name
            } else if (commands[i].origin_village_id === data.village_id) {
                commands[i].origin_village_name = data.name
                commands[i].originVillage.name = data.name
            }
        }

        eventQueue.trigger(eventTypeProvider.ATTACK_VIEW_VILLAGE_RENAMED, [data])
    }

    const onVillageSwitched = function (e, newVillageId) {
        if (filterParams[FILTER_TYPES.VILLAGE].length) {
            filterParams[FILTER_TYPES.VILLAGE] = [newVillageId]

            attackView.loadCommands()
        }
    }

    /**
     * @param {CommandModel} command
     * @return {String} Slowest unit
     */
    const getSlowestUnit = function (command) {
        const commandDuration = command.model.duration
        let units = {}
        const origin = { x: command.origin_x, y: command.origin_y }
        const target = { x: command.target_x, y: command.target_y }
        let travelTimes = []

        UNIT_SPEED_ORDER.forEach(function (unit) {
            units[unit] = 1

            travelTimes.push({
                unit: unit,
                duration: utils.getTravelTime(origin, target, units, command.command_type, {})
            })
        })

        travelTimes = travelTimes.map(function (travelTime) {
            travelTime.duration = Math.abs(travelTime.duration - commandDuration)
            return travelTime
        }).sort(function (a, b) {
            return a.duration - b.duration
        })

        return travelTimes[0].unit
    }

    /**
     * Sort a set of villages by distance from a specified village.
     *
     * @param {Array[{x: Number, y: Number}]} villages List of village that will be sorted.
     * @param {VillageModel} origin
     * @return {Array} Sorted villages
     */
    const sortByDistance = function (villages, origin) {
        return villages.sort(function (villageA, villageB) {
            let distA = math.actualDistance(origin, villageA)
            let distB = math.actualDistance(origin, villageB)

            return distA - distB
        })
    }

    /**
     * Order:
     * - Barbarian villages.
     * - Own villages.
     * - Tribe villages.
     *
     * @param {VillageModel} origin
     * @param {Function} callback
     */
    const closestNonHostileVillage = function (origin, callback) {
        const size = 25
        let loadBlockIndex = 0

        if (mapData.hasTownDataInChunk(origin.x, origin.y)) {
            const sectors = mapData.loadTownData(origin.x, origin.y, size, size, size)
            let targets = []
            let closestTargets
            const tribeId = modelDataService.getSelectedCharacter().getTribeId()
            const playerId = modelDataService.getSelectedCharacter().getId()

            sectors.forEach(function (sector) {
                for (let x in sector.data) {
                    for (let y in sector.data[x]) {
                        targets.push(sector.data[x][y])
                    }
                }
            })


            const barbs = targets.filter(function (target) {
                return target.character_id === null && target.id > 0
            })

            const own = targets.filter(function (target) {
                return target.character_id === playerId && origin.id !== target.id
            })

            if (tribeId) {
                const tribe = targets.filter(function (target) {
                    return tribeId && target.tribe_id === tribeId
                })
            }

            if (barbs.length) {
                closestTargets = sortByDistance(barbs, origin)
            } else if (own.length) {
                closestTargets = sortByDistance(own, origin)
            } else if (tribe.length) {
                closestTargets = sortByDistance(tribe, origin)
            } else {
                return callback(false)
            }

            return callback(closestTargets[0])
        }

        const loads = convert.scaledGridCoordinates(origin.x, origin.y, size, size, size)

        mapData.loadTownDataAsync(origin.x, origin.y, size, size, function () {
            if (++loadBlockIndex === loads.length) {
                closestNonHostileVillage(origin, callback)
            }
        })
    }

    /**
     * @param {Object} data The data-object from the backend
     */
    const onOverviewIncomming = function (data) {
        commands = data.commands

        for (let i = 0; i < commands.length; i++) {
            overviewService.formatCommand(commands[i])
            commands[i].slowestUnit = getSlowestUnit(commands[i])
        }

        commands = commands.filter(function (command) {
            return filters[FILTER_TYPES.INCOMING_UNITS][command.slowestUnit]
        })

        eventQueue.trigger(eventTypeProvider.ATTACK_VIEW_COMMANDS_LOADED, [commands])
    }

    let attackView = {}

    attackView.loadCommands = function () { 
        const incomingCommands = globalInfoModel.getCommandListModel().getIncomingCommands().length
        const count = incomingCommands > 25 ? incomingCommands : 25

        socketService.emit(routeProvider.OVERVIEW_GET_INCOMING, {
            'count': count,
            'offset': 0,
            'sorting': sorting.column,
            'reverse': sorting.reverse ? 1 : 0,
            'groups': [],
            'command_types': filterParams[FILTER_TYPES.COMMAND_TYPES],
            'villages': filterParams[FILTER_TYPES.VILLAGE]
        }, onOverviewIncomming)
    }

    attackView.getCommands = function () {
        return commands
    }

    attackView.getFilters = function () {
        return filters
    }

    attackView.getSortings = function () {
        return sorting
    }

    /**
     * Toggles the given filter.
     *
     * @param {string} type The category of the filter (see FILTER_TYPES)
     * @param {string} opt_filter The filter to be toggled.
     */
    attackView.toggleFilter = function (type, opt_filter) {
        if (!opt_filter) {
            filters[type] = !filters[type]
        } else {
            filters[type][opt_filter] = !filters[type][opt_filter]
        }

        // format filters for the backend
        formatFilters()
        Lockr.set(STORAGE_KEYS.FILTERS, filters)
        attackView.loadCommands()
    }

    attackView.toggleSorting = function (newColumn) {
        if (newColumn === sorting.column) {
            sorting.reverse = !sorting.reverse
        } else {
            sorting.column = newColumn
            sorting.reverse = false
        }

        attackView.loadCommands()
    }

    /**
     * Set an automatic command with all units from the village
     * and start the CommandQueue module if it's disabled.
     *
     * @param {Object} command Data of the command like origin, target.
     * @param {String} date Date that the command has to leave.
     */
    attackView.setCommander = function (command, date) {
        closestNonHostileVillage(command.targetVillage, function (closestVillage) {
            const origin = command.targetVillage
            const target = closestVillage
            const type = target.character_id === null ? 'attack' : 'support'

            commandQueue.addCommand({
                origin: origin,
                target: target,
                date: date,
                dateType: 'out',
                units: {
                    spear: '*',
                    sword: '*',
                    axe: '*',
                    archer: '*',
                    light_cavalry: '*',
                    mounted_archer: '*',
                    heavy_cavalry: '*',
                    ram: '*',
                    catapult: '*',
                    snob: '*',
                    knight: '*',
                    doppelsoldner: '*',
                    trebuchet: '*'
                },
                officers: {},
                type: type,
                catapultTarget: 'wall'
            })

            if (!commandQueue.isRunning()) {
                commandQueue.start()
            }
        })
    }

    attackView.commandQueueEnabled = function () {
        return !!commandQueue
    }

    attackView.isInitialized = function () {
        return initialized
    }

    attackView.init = function () {
        for (let i = 0; i < UNIT_SPEED_ORDER.length; i++) {
            INCOMING_UNITS_FILTER[UNIT_SPEED_ORDER[i]] = true
        }

        for (let i in COMMAND_TYPES) {
            COMMAND_TYPES_FILTER[COMMAND_TYPES[i]] = true
        }

        try {
            commandQueue = require('two/commandQueue')
        } catch (e) {}

        const defaultFilters = {
            [FILTER_TYPES.COMMAND_TYPES]: angular.copy(COMMAND_TYPES_FILTER),
            [FILTER_TYPES.INCOMING_UNITS]: angular.copy(INCOMING_UNITS_FILTER),
            [FILTER_TYPES.VILLAGE]: false
        }

        initialized = true
        globalInfoModel = modelDataService.getSelectedCharacter().getGlobalInfo()
        filters = Lockr.get(STORAGE_KEYS.FILTERS, defaultFilters, true)

        ready(function () {
            formatFilters()

            $rootScope.$on(eventTypeProvider.COMMAND_INCOMING, onCommandIncomming)
            $rootScope.$on(eventTypeProvider.COMMAND_CANCELLED, onCommandCancelled)
            $rootScope.$on(eventTypeProvider.MAP_SELECTED_VILLAGE, onVillageSwitched)
            $rootScope.$on(eventTypeProvider.VILLAGE_NAME_CHANGED, onVillageNameChanged)
            $rootScope.$on(eventTypeProvider.COMMAND_IGNORED, onCommandIgnored)

            attackView.loadCommands()
        }, ['initial_village'])
    }

    return attackView
})

define('two/attackView/events', [], function () {
    angular.extend(eventTypeProvider, {
        ATTACK_VIEW_FILTERS_CHANGED: 'attack_view_filters_changed',
        ATTACK_VIEW_SORTING_CHANGED: 'attack_view_sorting_changed',
        ATTACK_VIEW_COMMAND_CANCELLED: 'attack_view_command_cancelled',
        ATTACK_VIEW_COMMAND_IGNORED: 'attack_view_command_ignored',
        ATTACK_VIEW_VILLAGE_RENAMED: 'attack_view_village_renamed',
        ATTACK_VIEW_COMMANDS_LOADED: 'attack_view_commands_loaded'
    })
})

define('two/attackView/ui', [
    'two/ui',
    'two/attackView',
    'two/EventScope',
    'two/utils',
    'two/attackView/types/columns',
    'two/attackView/types/commands',
    'two/attackView/types/filters',
    'two/attackView/unitSpeedOrder',
    'conf/unitTypes',
    'queues/EventQueue',
    'helper/time',
    'battlecat'
], function (
    interfaceOverflow,
    attackView,
    EventScope,
    utils,
    COLUMN_TYPES,
    COMMAND_TYPES,
    FILTER_TYPES,
    UNIT_SPEED_ORDER,
    UNIT_TYPES,
    eventQueue,
    timeHelper,
    $
) {
    let $scope
    let $button

    const nowSeconds = function () {
        return Date.now() / 1000
    }

    const copyTimeModal = function (time) {
        let modalScope = $rootScope.$new()
        modalScope.text = $filter('readableDateFilter')(time * 1000, $rootScope.loc.ale, $rootScope.GAME_TIMEZONE, $rootScope.GAME_TIME_OFFSET, 'H:mm:ss:sss dd/MM/yyyy')
        modalScope.title = $filter('i18n')('copy', $rootScope.loc.ale, 'attack_view')
        windowManagerService.getModal('!twoverflow_attack_view_show_text_modal', modalScope)
    }

    const removeTroops = function (command) {
        const formatedDate = $filter('readableDateFilter')((command.time_completed - 10) * 1000, $rootScope.loc.ale, $rootScope.GAME_TIMEZONE, $rootScope.GAME_TIME_OFFSET, 'H:mm:ss:sss dd/MM/yyyy')
        attackView.setCommander(command, formatedDate)
    }

    const switchWindowSize = function () {
        let $window = $('#two-attack-view').parent()
        let $wrapper = $('#wrapper')

        $window.toggleClass('fullsize')
        $wrapper.toggleClass('window-fullsize')
    }

    const updateVisibileCommands = function () {
        const offset = $scope.pagination.offset
        const limit = $scope.pagination.limit

        $scope.visibleCommands = $scope.commands.slice(offset, offset + limit)
        $scope.pagination.count = $scope.commands.length
    }

    const checkCommands = function () {
        const now = Date.now()

        for (let i = 0; i < $scope.commands.length; i++) {
            if ($scope.commands[i].model.percent(now) === 100) {
                $scope.commands.splice(i, 1)
            }
        }

        updateVisibileCommands()
    }

    // scope functions

    const toggleFilter = function (type, _filter) {
        attackView.toggleFilter(type, _filter)
        $scope.filters = attackView.getFilters()
    }

    const toggleSorting = function (column) {
        attackView.toggleSorting(column)
        $scope.sorting = attackView.getSortings()
    }

    const eventHandlers = {
        updateCommands: function () {
            $scope.commands = attackView.getCommands()
        },
        onVillageSwitched: function () {
            $scope.selectedVillageId = modelDataService.getSelectedVillage().getId()
        }
    }

    const init = function () {
        $button = interfaceOverflow.addMenuButton('AttackView', 40)
        $button.addEventListener('click', buildWindow)

        interfaceOverflow.addTemplate('twoverflow_attack_view_main', `<div id=\"two-attack-view\" class=\"win-content two-window\">
																																									<header class=\"win-head\">
																																										<h2>AttackView</h2>
																																										<ul class=\"list-btn\">
																																											<li>
																																												<a href=\"#\" class=\"size-34x34 btn-orange icon-26x26-double-arrow\" ng-click=\"switchWindowSize()\"/>
																																												<li>
																																													<a href=\"#\" class=\"size-34x34 btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																												</ul>
																																											</header>
																																											<div class=\"win-main\" scrollbar=\"\">
																																												<div class=\"box-paper\">
																																													<div class=\"scroll-wrap rich-text\">
																																														<div class=\"filters\">
																																															<table class=\"tbl-border-light\">
																																																<tr>
																																																	<th>{{ 'village' | i18n:loc.ale:'common' }}<tr>
																																																			<td>
																																																				<div class=\"box-border-dark icon\" ng-class=\"{'active': filters[FILTER_TYPES.VILLAGE]}\" ng-click=\"toggleFilter(FILTER_TYPES.VILLAGE)\" tooltip=\"\" tooltip-content=\"{{ 'current_only_tooltip' | i18n:loc.ale:'attack_view' }}\">
																																																					<span class=\"icon-34x34-village-info icon-bg-black\"/>
																																																				</div>
																																																			</table>
																																																			<table class=\"tbl-border-light\">
																																																				<tr>
																																																					<th>{{ 'filter_types' | i18n:loc.ale:'attack_view' }}<tr>
																																																							<td>
																																																								<div class=\"box-border-dark icon\" ng-class=\"{'active': filters[FILTER_TYPES.COMMAND_TYPES][COMMAND_TYPES.ATTACK]}\" ng-click=\"toggleFilter(FILTER_TYPES.COMMAND_TYPES, COMMAND_TYPES.ATTACK)\" tooltip=\"\" tooltip-content=\"{{ 'filter_show_attacks_tooltip' | i18n:loc.ale:'attack_view' }}\">
																																																									<span class=\"icon-34x34-attack icon-bg-black\"/>
																																																								</div>
																																																								<div class=\"box-border-dark icon\" ng-class=\"{'active': filters[FILTER_TYPES.COMMAND_TYPES][COMMAND_TYPES.SUPPORT]}\" ng-click=\"toggleFilter(FILTER_TYPES.COMMAND_TYPES, COMMAND_TYPES.SUPPORT)\" tooltip=\"\" tooltip-content=\"{{ 'filter_show_supports_tooltip' | i18n:loc.ale:'attack_view' }}\">
																																																									<span class=\"icon-34x34-support icon-bg-black\"/>
																																																								</div>
																																																								<div class=\"box-border-dark icon\" ng-class=\"{'active': filters[FILTER_TYPES.COMMAND_TYPES][COMMAND_TYPES.RELOCATE]}\" ng-click=\"toggleFilter(FILTER_TYPES.COMMAND_TYPES, COMMAND_TYPES.RELOCATE)\" tooltip=\"\" tooltip-content=\"{{ 'filter_show_relocations_tooltip' | i18n:loc.ale:'attack_view' }}\">
																																																									<span class=\"icon-34x34-relocate icon-bg-black\"/>
																																																								</div>
																																																							</table>
																																																							<table class=\"tbl-border-light\">
																																																								<tr>
																																																									<th>{{ 'filter_incoming_units' | i18n:loc.ale:'attack_view' }}<tr>
																																																											<td>
																																																												<div ng-repeat=\"unit in ::UNIT_SPEED_ORDER\" class=\"box-border-dark icon\" ng-class=\"{'active': filters[FILTER_TYPES.INCOMING_UNITS][unit]}\" ng-click=\"toggleFilter(FILTER_TYPES.INCOMING_UNITS, unit)\" tooltip=\"\" tooltip-content=\"{{ unit | i18n:loc.ale:'unit_names' }}\">
																																																													<span class=\"icon-34x34-unit-{{ unit }} icon-bg-black\"/>
																																																												</div>
																																																											</table>
																																																										</div>
																																																										<div class=\"page-wrap\" pagination=\"pagination\"/>
																																																										<p class=\"text-center\" ng-show=\"!visibleCommands.length\">{{ 'no_incoming' | i18n:loc.ale:'attack_view' }}<table class=\"tbl-border-light commands-table\" ng-show=\"visibleCommands.length\">
																																																												<col width=\"8%\">
																																																													<col width=\"14%\">
																																																														<col>
																																																															<col>
																																																																<col width=\"4%\">
																																																																	<col width=\"15%\">
																																																																		<col width=\"11%\">
																																																																			<thead class=\"sorting\">
																																																																				<tr>
																																																																					<th ng-click=\"toggleSorting(COLUMN_TYPES.COMMAND_TYPE)\" tooltip=\"\" tooltip-content=\"{{ 'command_type_tooltip' | i18n:loc.ale:'attack_view' }}\">{{ 'command_type' | i18n:loc.ale:'attack_view' }} <span class=\"arrow\" ng-show=\"sorting.column == COLUMN_TYPES.COMMAND_TYPE\" ng-class=\"{'icon-26x26-normal-arrow-down': sorting.reverse, 'icon-26x26-normal-arrow-up': !sorting.reverse}\"/>
																																																																						<th ng-click=\"toggleSorting(COLUMN_TYPES.ORIGIN_CHARACTER)\">{{ 'player' | i18n:loc.ale:'common' }} <span class=\"arrow\" ng-show=\"sorting.column == COLUMN_TYPES.ORIGIN_CHARACTER\" ng-class=\"{'icon-26x26-normal-arrow-down': sorting.reverse, 'icon-26x26-normal-arrow-up': !sorting.reverse}\"/>
																																																																							<th ng-click=\"toggleSorting(COLUMN_TYPES.ORIGIN_VILLAGE)\">{{ 'origin' | i18n:loc.ale:'common' }} <span class=\"arrow\" ng-show=\"sorting.column == COLUMN_TYPES.ORIGIN_VILLAGE\" ng-class=\"{'icon-26x26-normal-arrow-down': sorting.reverse, 'icon-26x26-normal-arrow-up': !sorting.reverse}\"/>
																																																																								<th ng-click=\"toggleSorting(COLUMN_TYPES.TARGET_VILLAGE)\">{{ 'target' | i18n:loc.ale:'common' }} <span class=\"arrow\" ng-show=\"sorting.column == COLUMN_TYPES.TARGET_VILLAGE\" ng-class=\"{'icon-26x26-normal-arrow-down': sorting.reverse, 'icon-26x26-normal-arrow-up': !sorting.reverse}\"/>
																																																																									<th tooltip=\"\" tooltip-content=\"{{ 'slowest_unit_tooltip' | i18n:loc.ale:'attack_view' }}\">{{ 'slowest_unit' | i18n:loc.ale:'attack_view' }}<th ng-click=\"toggleSorting(COLUMN_TYPES.TIME_COMPLETED)\">{{ 'arrive' | i18n:loc.ale:'common' }} <span class=\"arrow\" ng-show=\"sorting.column == COLUMN_TYPES.TIME_COMPLETED\" ng-class=\"{'icon-26x26-normal-arrow-down': sorting.reverse, 'icon-26x26-normal-arrow-up': !sorting.reverse}\"/>
																																																																											<th>{{ 'actions' | i18n:loc.ale:'attack_view' }}<tbody>
																																																																													<tr ng-repeat=\"command in visibleCommands\" class=\"{{ command.command_type }}\" ng-class=\"{'snob': command.slowestUnit === UNIT_TYPES.SNOB}\">
																																																																														<td>
																																																																															<span class=\"icon-20x20-{{ command.command_type }}\"/>
																																																																															<td ng-click=\"openCharacterProfile(command.originCharacter.id)\" class=\"character\">
																																																																																<span class=\"name\">{{ command.originCharacter.name }}</span>
																																																																																<td ng-class=\"{'selected': command.originVillage.id === selectedVillageId}\" class=\"village\">
																																																																																	<span class=\"name\" ng-click=\"openVillageInfo(command.originVillage.id)\">{{ command.originVillage.name }}</span>
																																																																																	<span class=\"coords\" ng-click=\"jumpToVillage(command.originVillage.x, command.originVillage.y)\">({{ command.originVillage.x }}|{{ command.originVillage.y }})</span>
																																																																																	<td ng-class=\"{'selected': command.targetVillage.id === selectedVillageId}\" class=\"village\">
																																																																																		<span class=\"name\" ng-click=\"openVillageInfo(command.targetVillage.id)\">{{ command.targetVillage.name }}</span>
																																																																																		<span class=\"coords\" ng-click=\"jumpToVillage(command.targetVillage.x, command.targetVillage.y)\">({{ command.targetVillage.x }}|{{ command.targetVillage.y }})</span>
																																																																																		<td>
																																																																																			<span class=\"icon-20x20-unit-{{ command.slowestUnit }}\"/>
																																																																																			<td>
																																																																																				<div class=\"progress-wrapper\" tooltip=\"\" tooltip-content=\"{{ command.model.arrivalTime() | readableDateFilter:loc.ale:GAME_TIMEZONE:GAME_TIME_OFFSET }}\">
																																																																																					<div class=\"progress-bar\" ng-style=\"{width: command.model.percent() + '%'}\"/>
																																																																																					<div class=\"progress-text\">
																																																																																						<span>{{ command.model.countdown() }}</span>
																																																																																					</div>
																																																																																				</div>
																																																																																				<td>
																																																																																					<a ng-click=\"copyTimeModal(command.time_completed)\" class=\"btn btn-orange size-20x20 icon-20x20-arrivetime\" tooltip=\"\" tooltip-content=\"{{ 'commands_copy_arrival_tooltip' | i18n:loc.ale:'attack_view' }}\"/>
																																																																																					<a ng-click=\"copyTimeModal(command.time_completed + (command.time_completed - command.time_start))\" class=\"btn btn-red size-20x20 icon-20x20-backtime\" tooltip=\"\" tooltip-content=\"{{ 'commands_copy_backtime_tooltip' | i18n:loc.ale:'attack_view' }}\"/>
																																																																																					<a ng-if=\"commandQueueEnabled\" ng-click=\"removeTroops(command)\" class=\"btn btn-orange size-20x20 icon-20x20-units-outgoing\" tooltip=\"\" tooltip-content=\"{{ 'commands_set_remove_tooltip' | i18n:loc.ale:'attack_view' }}\"/>
																																																																																				</table>
																																																																																				<div class=\"page-wrap\" pagination=\"pagination\"/>
																																																																																			</div>
																																																																																		</div>
																																																																																	</div>
																																																																																</div>`)
        interfaceOverflow.addTemplate('twoverflow_attack_view_show_text_modal', `<div id=\"show-text-modal\" class=\"win-content\">
																																																																																	<header class=\"win-head\">
																																																																																		<h3>{{ title }}</h3>
																																																																																		<ul class=\"list-btn sprite\">
																																																																																			<li>
																																																																																				<a href=\"#\" class=\"btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																			</ul>
																																																																																		</header>
																																																																																		<div class=\"win-main\" scrollbar=\"\">
																																																																																			<div class=\"box-paper\">
																																																																																				<div class=\"scroll-wrap\">
																																																																																					<form ng-submit=\"closeWindow()\">
																																																																																						<input class=\"input-border text-center\" ng-model=\"text\">
																																																																																						</form>
																																																																																					</div>
																																																																																				</div>
																																																																																			</div>
																																																																																			<footer class=\"win-foot sprite-fill\">
																																																																																				<ul class=\"list-btn list-center\">
																																																																																					<li>
																																																																																						<a href=\"#\" class=\"btn-green btn-border\" ng-click=\"closeWindow()\">OK</a>
																																																																																					</ul>
																																																																																				</footer>
																																																																																			</div>`)
        interfaceOverflow.addStyle('#two-attack-view table.commands-table{table-layout:fixed;font-size:13px;margin-bottom:10px}#two-attack-view table.commands-table th{text-align:center;padding:0px}#two-attack-view table.commands-table td{padding:1px 0;min-height:initial;border:none;text-align:center}#two-attack-view table.commands-table tr.attack.snob td{background:#bb8658}#two-attack-view table.commands-table tr.support td,#two-attack-view table.commands-table tr.relocate td{background:#9c9368}#two-attack-view table.commands-table .empty td{height:32px}#two-attack-view table.commands-table .sorting .arrow{margin-top:-4px}#two-attack-view .village .coords{font-size:11px;color:#71471a}#two-attack-view .village .coords:hover{color:#ffde00;text-shadow:0 1px 0 #000}#two-attack-view .village .name:hover{color:#fff;text-shadow:0 1px 0 #000}#two-attack-view .village.selected .name{font-weight:bold}#two-attack-view .character .name:hover{color:#fff;text-shadow:1px 1px 0 #000}#two-attack-view .progress-wrapper{height:20px;margin-bottom:0}#two-attack-view .progress-wrapper .progress-text{position:absolute;width:100%;height:100%;text-align:center;z-index:10;padding:0 5px;line-height:20px;color:#f0ffc9;overflow:hidden}#two-attack-view .filters{height:95px;margin-bottom:10px}#two-attack-view .filters table{width:auto;float:left;margin:5px}#two-attack-view .filters .icon{width:38px;float:left;margin:0 6px}#two-attack-view .filters .icon.active:before{box-shadow:0 0 0 1px #000,-1px -1px 0 2px #ac9c44,0 0 0 3px #ac9c44,0 0 0 4px #000;border-radius:1px;content:"";position:absolute;width:38px;height:38px;left:-1px;top:-1px}#two-attack-view .filters td{padding:6px}#two-attack-view .icon-20x20-backtime{background-image:url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABQAAAAUCAYAAACNiR0NAAAABmJLR0QA/wD/AP+gvaeTAAAEMklEQVQ4y42US2xUdRTGf3funZn/PHqnnVdpKZZ2RCWBVESgoZogSAKKEEAlGhVNLMGg0QiJKxYudIdoTEyDj8SFGo2seDUGhEQqRHk/UimDpdAptHMr8+jM3Dv35QJbi9KEszzJ+eU753z5JKYuOQGBUpAa2SLiuPgBPBKGrZAPlSlmoQLYk4ekqUCmEHHL0pslRb7fsNwWF8L/DIz5Fanftey0oogBr65rk8HS3WC6jyY8ckfZdNtfWdX++tzGIDMabAJmArte4my/l/c//vaLoFc6jmP3iCqD41B5Mi0BId1Hk+V6ljfEQlvWL2xZoY/lKOTLGCY01tZhVLMkRJEtqzoeyUvSnN70SNZRXC1iUylDVZmszhQiDmbH9Lrgpta4mKPlCjy95D6Wrn8GAKFEEfEmdG2Qowd+4I0XFrUC7+w7eL5sCu8hdL3imaQuYFl6c9l021vjYk7Y72Xjq4/z1IaNCCVKMRckq+moiQDJ2bN48uV3GbnSx9b1ra1l0223LL05AYF/Vw4S80jyonnN6paq5YTe3LyU2rpaYrFpJGfPItlcTzI1H8R8cC38NTFiaojhSzeJJ8KNJ/4YOmP43GsTCmWLiGG5LTUBb2LuzGm3e3Ij3321m5Hey6A0AVAcPjmhQcSbuDyU5sF6e5phuS2yRWQC6Lj4x62h1vjJ3BwjlUoiYn52ffolmUtnuXj4ADu2b7/DFoN9RVQ1gAthx8U/+Sk4LiGAQtFAHzXIajpr16yiu/tX98euzyWAzrc6Abj8+1G0TIZ8uYx/xJpgjANlWfEKqjaZbIlixQQgdDHDyuULWLFisZTVdBJxQTIVA2uQ+qZ6KoU0nhqV09f+QoIxj4ThAWRVJWLZToNXUaarYR8Hdm+iZBic7N5LbmgI0xclERcAFLIVAHRtkFOHjwBwNHNryK9I/bZCXlFVIk6ZuSbukidmR1Z+/cliAHzRBjKjBTq37bz9gEAAgA+2vQjAjb4j9F6pUCga/Hzm5v6A5KRDFkXF1UnWRcRj256d/vam9zrJXT0GwGc7V+ONRwAwtTwAa9bs4ND+PTy8MMW5az7+vJ7lXKZ4IeiVjsuIgaylVxTHxf/S84+u3bh5Mbmrx/D6Y1hjGtaYBjduH9g0RonNSmH4o/T1j9JzeoBixSRbsi9ktNIuRXJ6vFVbA2ypVoiZNuay+qj62r6u1R0ee4i65Iw7rDEOnLegC4CSqwxf18b23C0cFMenF5wKJzLZfLDtuW/4pWt1Ry6XY8/ug8jRB6gN3GI0k6VtXcq9csvqtm2rTyjS+YDkpGXEgLdq/z++EhA2hYjbmMtMx7P8+4/Wbdj64U89/cP5Xlli2HGcUsAnjziulMGxbrheRu4lYH21QjSarvXQoraZbQC/nUoflzwMyx6hVz26MRVkysROQNhQ8XmqQr1XwH/rb2Du69Eebp25AAAAAElFTkSuQmCC")}#two-attack-view .icon-20x20-arrivetime{background-image:url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABQAAAAUCAYAAACNiR0NAAAABmJLR0QA/wD/AP+gvaeTAAAEW0lEQVQ4y4WUWWxUZRiGn7PMnNPOVtvODHQBSlulAUFBoQiEaBHBhCsSFaIhIe6JSyAkRkO8NpErY2KoYuINISkkRFAjEUyAUCQsBSu1BVpKZ2DmTNuZzsyZMz3L70Vbgkjqe/Ul//89//K9eSX+KyUKFcVKQopDxBNoALJE2VXJBUzyBpQA9xG9SA+DbF2vdRxrvqQqLWVHNAkITm8saKo0KBz3hqrqt32WlXkUWHoQZvlpQFbWmLZo//zj7W8ua7JRUoKSz+DOXYVrSZMfjnV/W+mTuvHcs/okIw9DFYAoBCw/DY6QX9yycemer9/p6KiQE7ilIj4vwNXBFIO3M1iFLKta4suNvLUwZzpZTxWZiEvJhMkHgYpf1+cKSazfsnHpnve2rVqYTg2xdvMrPL76JWKNNSxesYB1LyyDiQQ9fWkCmhxzkRuLZTcpVC1lOU4eEDNPDUzitJVc6eUDn6zuSAwl2PDGLqrnx9ECPob6kkxaPiLBEK1LniIaFVz/c4SAJsf6U2ZaEfZwxMOYuaVCJTWypKz68LXV7y6sigWf7thMdfMKkMOgryA2r5pYYwWBaA3FzBhFM8uiRXFOnumn/jGt0SjYl8t+MWzbFABkxSFSdkTTE3F3zkDyBnptw/2J5VMXpwq1gfT1AQ4eOIyi1AHw5II5hCp80bIjmhSHyEyP7Ak0AcFwuIKR/vy/PLVv7156T/1M4u8e9n/1HXqNRnNzjMS9AuGQBlMfF5zxKoA6U2hph5xp0nv+ErX1KVqfXctbH+yk65tOAOa1tolNm56TjIyFNVpmIl8GwBMEHnSzKkuUJUHh8vAYcihMIFQi3hAHZ4T65hq27dyKkbGI1uqS7a/mXO8F+gZGuDZ0j4nClFsU1adj2wrgyq5KTlOlwTOJ8STApVO/Y2VGAJgwSgBEa3VsfzXZZJKLvxyjWC7z8+G3CQf9+FS13nG9ueEwEUBRqmywEfrAvWLF4rqq5fmiwCvcIjuqYCTu8v5nnXQd7+bgoZ/48dduXF8F4ZpaNj0/j60bgly+YLTeNMyUYosxPUhONaBUpeq3K7G7T/Ym2pfWh5ZU1MzBX/0XV/64iVYe4+jR3QD4aqeGaWdylPNjABw9upv9X3R+9GVXwsjmrZQCiJDjOI4scjnTyZZc0ZhKJmM9PcNYlsu4CLJjez3jt65ij45jpZPYhVG8SRNFrcQc7eeZ9evIl9xI96Xh4yqAAaXoJCOW3zuRGjfNwbRob6wNbkkYxTizaDx9B0+pY93rnWdTYxPf+xQ9p0yvCRPciEtJqFpKEfZwyXaupArOYLbM+JK2lS3HDhyRbgwanO6eoPvEaWLxOixLY+WOrrP5onUI4Z2TdMeQZgtYySaGrM6VJVFfmnRjsiwHXEG8KR5p2/fpxjWv7jpyyCd7JxR8v03nY0Fidt2H+z1dcz1LFx7xlctb2gHO9wz1+CS1L2tZSabD4f+Asx7g+a0JbYJJg6lgAPgHUh4QWRIJr4EAAAAASUVORK5CYII=")}')
    }

    const buildWindow = function () {
        $scope = $rootScope.$new()
        $scope.commandQueueEnabled = attackView.commandQueueEnabled()
        $scope.commands = attackView.getCommands()
        $scope.selectedVillageId = modelDataService.getSelectedVillage().getId()
        $scope.filters = attackView.getFilters()
        $scope.sorting = attackView.getSortings()
        $scope.UNIT_TYPES = UNIT_TYPES
        $scope.FILTER_TYPES = FILTER_TYPES
        $scope.COMMAND_TYPES = COMMAND_TYPES
        $scope.UNIT_SPEED_ORDER = UNIT_SPEED_ORDER
        $scope.COLUMN_TYPES = COLUMN_TYPES
        $scope.pagination = {
            count: $scope.commands.length,
            offset: 0,
            loader: updateVisibileCommands,
            limit: storageService.getPaginationLimit()
        }

        // functions
        $scope.openCharacterProfile = windowDisplayService.openCharacterProfile
        $scope.openVillageInfo = windowDisplayService.openVillageInfo
        $scope.jumpToVillage = mapService.jumpToVillage
        $scope.now = nowSeconds
        $scope.copyTimeModal = copyTimeModal
        $scope.removeTroops = removeTroops
        $scope.switchWindowSize = switchWindowSize
        $scope.toggleFilter = toggleFilter
        $scope.toggleSorting = toggleSorting

        updateVisibileCommands()

        let eventScope = new EventScope('twoverflow_queue_window', function onWindowClose() {
            timeHelper.timer.remove(checkCommands)
        })
        eventScope.register(eventTypeProvider.MAP_SELECTED_VILLAGE, eventHandlers.onVillageSwitched, true)
        eventScope.register(eventTypeProvider.ATTACK_VIEW_COMMANDS_LOADED, eventHandlers.updateCommands)
        eventScope.register(eventTypeProvider.ATTACK_VIEW_COMMAND_CANCELLED, eventHandlers.updateCommands)
        eventScope.register(eventTypeProvider.ATTACK_VIEW_COMMAND_IGNORED, eventHandlers.updateCommands)
        eventScope.register(eventTypeProvider.ATTACK_VIEW_VILLAGE_RENAMED, eventHandlers.updateCommands)

        windowManagerService.getScreenWithInjectedScope('!twoverflow_attack_view_main', $scope)

        timeHelper.timer.add(checkCommands)
    }

    return init
})

define('two/attackView/types/columns', [], function () {
    return {
        'ORIGIN_VILLAGE': 'origin_village_name',
        'COMMAND_TYPE': 'command_type',
        'TARGET_VILLAGE': 'target_village_name',
        'TIME_COMPLETED': 'time_completed',
        'ORIGIN_CHARACTER': 'origin_character_name'
    }
})

define('two/attackView/types/commands', [], function () {
    return {
        'ATTACK': 'attack',
        'SUPPORT': 'support',
        'RELOCATE': 'relocate'
    }
})

define('two/attackView/types/filters', [], function () {
    return {
        'COMMAND_TYPES' : 'command_types',
        'VILLAGE' : 'village',
        'INCOMING_UNITS' : 'incoming_units'
    }
})

define('two/attackView/unitSpeedOrder', [
    'conf/unitTypes'
], function (
    UNIT_TYPES
) {
    return [
        UNIT_TYPES.LIGHT_CAVALRY,
        UNIT_TYPES.HEAVY_CAVALRY,
        UNIT_TYPES.AXE,
        UNIT_TYPES.SWORD,
        UNIT_TYPES.RAM,
        UNIT_TYPES.SNOB,
        UNIT_TYPES.TREBUCHET
    ]
})

require([
    'two/language',
    'two/ready',
    'two/attackView',
    'two/attackView/ui',
    'two/attackView/events'
], function (
    twoLanguage,
    ready,
    attackView,
    attackViewInterface
) {
    if (attackView.isInitialized()) {
        return false
    }

    ready(function () {
        twoLanguage.add('attack_view', {"de_de":{"attack_view":{"filter_types":"Typen","filter_show_attacks_tooltip":"Zeige Angriffe","filter_show_supports_tooltip":"Zeige Unterst체tzungen","filter_show_relocations_tooltip":"Zeige Umsiedlungen","filter_incoming_units":"Eintreffende Einheiten","commands_copy_arrival_tooltip":"Kopiere Ankunftsdatum.","commands_copy_backtime_tooltip":"Kopiere Datum der R체ckkehr.","commands_set_remove_tooltip":"Erzeuge eine Befehlskette, um alle Truppen vor dem Angriff zu versenden.","command_type_tooltip":"Befehlstyp","slowest_unit_tooltip":"Langsamste Einheit","command_type":"AT","slowest_unit":"LE","actions":"Aktionen","no_incoming":"Keine eingehenden Befehle.","copy":"Kopiere","current_only_tooltip":"Nur aktuelles Dorf"}},"en_us":{"attack_view":{"filter_types":"Types","filter_show_attacks_tooltip":"Show attacks","filter_show_supports_tooltip":"Show supports","filter_show_relocations_tooltip":"Show relocations","filter_incoming_units":"Incoming Units","commands_copy_arrival_tooltip":"Copy arrival date.","commands_copy_backtime_tooltip":"Copy backtime date.","commands_set_remove_tooltip":"Set a CommandQueue to remove all troops before the attack hit.","command_type_tooltip":"Command Type","slowest_unit_tooltip":"Slowest Unit","command_type":"CT","slowest_unit":"SU","actions":"Actions","no_incoming":"No commands incoming.","copy":"Copy","current_only_tooltip":"Current village only"}},"pl_pl":{"attack_view":{"filter_types":"Rodzaj","filter_show_attacks_tooltip":"Poka탉 ataki","filter_show_supports_tooltip":"Poka탉 wsparcia","filter_show_relocations_tooltip":"Poka탉 przeniesienia","filter_incoming_units":"Nadchodz훳ce jednostki","commands_copy_arrival_tooltip":"Kopiuj czas dotarcia.","commands_copy_backtime_tooltip":"Kopiuj czas powrotu do wioski 탄r처d흢owej.","commands_set_remove_tooltip":"Wstaw rozkaz wycofania wojsk przed dotarciem ataku do Kolejki rozkaz처w.","command_type_tooltip":"Rodzaj","slowest_unit_tooltip":"Najwolniejsza jednostka","command_type":"Rodzaj","slowest_unit":"Co?","actions":"Dost휌pne akcje","no_incoming":"Brak nadchodz훳cych wojsk.","copy":"Copy","current_only_tooltip":"Tylko aktywna wioska"}},"pt_br":{"attack_view":{"filter_types":"Tipos","filter_show_attacks_tooltip":"Mostrar ataques","filter_show_supports_tooltip":"Mostrar apoios","filter_show_relocations_tooltip":"Mostrar transfer챗ncias","filter_incoming_units":"Unidades Chegando","commands_copy_arrival_tooltip":"Copiar data de chegada.","commands_copy_backtime_tooltip":"Copiar backtime.","commands_set_remove_tooltip":"Criar um comando no CommandQueue para remover todas tropas da aldeia antes do comando bater na aldeia.","command_type_tooltip":"Tipo de Comando","slowest_unit_tooltip":"Unidade mais Lenta","command_type":"TC","slowest_unit":"UL","actions":"A챌천es","no_incoming":"Nenhum comando chegando.","copy":"Copiar","current_only_tooltip":"Apenas aldeia selecionada"}},"ru_ru":{"attack_view":{"filter_types":"龜龜極�","filter_show_attacks_tooltip":"�棘克逵鈞逵�� 逵�逵克龜","filter_show_supports_tooltip":"�棘克逵鈞逵�� 極棘畇克�筠極剋筠戟龜�","filter_show_relocations_tooltip":"�棘克逵鈞逵�� 極筠�筠劇筠�筠戟龜�","filter_incoming_units":"��棘畇��龜筠 勻棘橘�克逵","commands_copy_arrival_tooltip":"�棘極龜�棘勻逵�� 畇逵�� 極�龜閨��龜�.","commands_copy_backtime_tooltip":"�棘極龜�棘勻逵�� 畇逵�� 勻棘鈞勻�逵�筠戟龜�.","commands_set_remove_tooltip":"Set a CommandQueue to remove all troops before the attack hit.","command_type_tooltip":"龜龜極 克棘劇逵戟畇�","slowest_unit_tooltip":"鬼逵劇�橘 劇筠畇剋筠戟戟�橘 勻棘龜戟","command_type":"CT","slowest_unit":"SU","actions":"�筠橘��勻龜�","no_incoming":"�筠� 勻�棘畇��龜� 克棘劇逵戟畇.","copy":"�棘極龜�棘勻逵��","current_only_tooltip":"龜棘剋�克棘 �筠克��逵� 畇筠�筠勻戟�"}}})
        attackView.init()
        attackViewInterface()
    })
})

define('two/autoCollector', [
    'queues/EventQueue',
    'helper/time',
    'Lockr'
], function (
    eventQueue,
    $timeHelper,
    Lockr
) {
    let initialized = false
    let running = false

    /**
     * Permite que o evento RESOURCE_DEPOSIT_JOB_COLLECTIBLE seja executado
     * apenas uma vez.
     */
    let recall = true

    /**
     * Next automatic reroll setTimeout ID.
     */
    let nextUpdateId = 0

    /**
     * Inicia um trabalho.
     *
     * @param {Object} job - Dados do trabalho
     */
    const startJob = function (job) {
        socketService.emit(routeProvider.RESOURCE_DEPOSIT_START_JOB, {
            job_id: job.id
        })
    }

    /**
     * Coleta um trabalho.
     *
     * @param {Object} job - Dados do trabalho
     */
    const finalizeJob = function (job) {
        socketService.emit(routeProvider.RESOURCE_DEPOSIT_COLLECT, {
            job_id: job.id,
            village_id: modelDataService.getSelectedVillage().getId()
        })
    }

    /**
     * For챌a a atualiza챌찾o das informa챌천es do dep처sito.
     */
    const updateDepositInfo = function () {
        socketService.emit(routeProvider.RESOURCE_DEPOSIT_GET_INFO, {})
    }

    /**
     * Faz a analise dos trabalhos sempre que um evento relacionado ao dep처sito
     * 챕 disparado.
     */
    const analyse = function () {
        if (!running) {
            return false
        }

        let data = modelDataService.getSelectedCharacter().getResourceDeposit()

        if (!data) {
            return false
        }

        if (data.getCurrentJob()) {
            return false
        }

        let collectible = data.getCollectibleJobs()

        if (collectible) {
            return finalizeJob(collectible.shift())
        }

        let ready = data.getReadyJobs()

        if (ready) {
            return startJob(getFastestJob(ready))
        }
    }

    /**
     * Obtem o trabalho de menor dura챌찾o.
     *
     * @param {Array} jobs - Lista de trabalhos prontos para serem iniciados.
     */
    const getFastestJob = function (jobs) {
        const sorted = jobs.sort(function (a, b) {
            return a.duration - b.duration
        })

        return sorted[0]
    }

    /**
     * Atualiza o timeout para que seja for챌ado a atualiza챌찾o das informa챌천es
     * do dep처sito quando for resetado.
     * Motivo: s처 챕 chamado automaticamente quando um milestone 챕 resetado,
     * e n찾o o di찼rio.
     * 
     * @param {Object} data - Os dados recebidos de RESOURCE_DEPOSIT_INFO
     */
    const rerollUpdater = function (data) {
        const timeLeft = data.time_next_reset * 1000 - Date.now() + 1000

        clearTimeout(nextUpdateId)
        nextUpdateId = setTimeout(updateDepositInfo, timeLeft)
    }

    /**
     * M챕todos p첬blicos do AutoCollector.
     *
     * @type {Object}
     */
    let autoCollector = {}

    /**
     * Inicializa o AutoDepois, configura os eventos.
     */
    autoCollector.init = function () {
        initialized = true

        $rootScope.$on(eventTypeProvider.RESOURCE_DEPOSIT_JOB_COLLECTIBLE, function () {
            if (!recall || !running) {
                return false
            }

            recall = false

            setTimeout(function () {
                recall = true
                analyse()
            }, 1500)
        })

        $rootScope.$on(eventTypeProvider.RESOURCE_DEPOSIT_JOBS_REROLLED, analyse)
        $rootScope.$on(eventTypeProvider.RESOURCE_DEPOSIT_JOB_COLLECTED, analyse)
        $rootScope.$on(eventTypeProvider.RESOURCE_DEPOSIT_INFO, function (event, data) {
            analyse()
            rerollUpdater(data)
        })
    }

    /**
     * Inicia a analise dos trabalhos.
     */
    autoCollector.start = function () {
        eventQueue.trigger(eventTypeProvider.AUTO_COLLECTOR_STARTED)
        running = true
        analyse()
    }

    /**
     * Para a analise dos trabalhos.
     */
    autoCollector.stop = function () {
        eventQueue.trigger(eventTypeProvider.AUTO_COLLECTOR_STOPPED)
        running = false
    }

    /**
     * Retorna se o modulo est찼 em funcionamento.
     */
    autoCollector.isRunning = function () {
        return running
    }

    /**
     * Retorna se o modulo est찼 inicializado.
     */
    autoCollector.isInitialized = function () {
        return initialized
    }

    return autoCollector
})

define('two/autoCollector/events', [], function () {
    angular.extend(eventTypeProvider, {
        AUTO_COLLECTOR_STARTED: 'auto_collector_started',
        AUTO_COLLECTOR_STOPPED: 'auto_collector_stopped',
        AUTO_COLLECTOR_SECONDVILLAGE_STARTED: 'auto_collector_secondvillage_started',
        AUTO_COLLECTOR_SECONDVILLAGE_STOPPED: 'auto_collector_secondvillage_stopped'
    })
})

define('two/autoCollector/ui', [
    'two/ui',
    'two/autoCollector',
    'two/utils',
    'queues/EventQueue'
], function (
    interfaceOverflow,
    autoCollector,
    utils,
    eventQueue
) {
    let $button

    const init = function () {
        $button = interfaceOverflow.addMenuButton('Collector', 50, $filter('i18n')('description', $rootScope.loc.ale, 'auto_collector'))

        $button.addEventListener('click', function () {
            if (autoCollector.isRunning()) {
                autoCollector.stop()
                autoCollector.secondVillage.stop()
                utils.emitNotif('success', $filter('i18n')('deactivated', $rootScope.loc.ale, 'auto_collector'))
            } else {
                autoCollector.start()
                autoCollector.secondVillage.start()
                utils.emitNotif('success', $filter('i18n')('activated', $rootScope.loc.ale, 'auto_collector'))
            }
        })

        eventQueue.register(eventTypeProvider.AUTO_COLLECTOR_STARTED, function () {
            $button.classList.remove('btn-green')
            $button.classList.add('btn-red')
        })

        eventQueue.register(eventTypeProvider.AUTO_COLLECTOR_STOPPED, function () {
            $button.classList.remove('btn-red')
            $button.classList.add('btn-green')
        })

        if (autoCollector.isRunning()) {
            eventQueue.trigger(eventTypeProvider.AUTO_COLLECTOR_STARTED)
        }

        return opener
    }

    return init
})

define('two/autoCollector/secondVillage', [
    'two/autoCollector',
    'queues/EventQueue',
    'helper/time',
    'models/SecondVillageModel'
], function (
    autoCollector,
    eventQueue,
    $timeHelper,
    SecondVillageModel
) {
    let initialized = false
    let running = false
    let secondVillageService = injector.get('secondVillageService')

    const getRunningJob = function (jobs) {
        const now = Date.now()

        for (let id in jobs) {
            if (jobs[id].time_started && jobs[id].time_completed) {
                if (now < $timeHelper.server2ClientTime(jobs[id].time_completed)) {
                    return jobs[id]
                }
            }
        }

        return false
    }

    const getCollectibleJob = function (jobs) {
        const now = Date.now()

        for (let id in jobs) {
            if (jobs[id].time_started && jobs[id].time_completed) {
                if ((now >= $timeHelper.server2ClientTime(jobs[id].time_completed)) && !jobs[id].collected) {
                    return id
                }
            }
        }

        return false
    }

    const finalizeJob = function (jobId) {
        socketService.emit(routeProvider.SECOND_VILLAGE_COLLECT_JOB_REWARD, {
            village_id: modelDataService.getSelectedVillage().getId(),
            job_id: jobId
        })
    }

    const startJob = function (jobId, callback) {
        socketService.emit(routeProvider.SECOND_VILLAGE_START_JOB, {
            village_id: modelDataService.getSelectedVillage().getId(),
            job_id: jobId
        }, callback)
    }

    const getFirstJob = function (jobs) {
        for (let id in jobs) {
            return id
        }

        return false
    }

    const updateSecondVillageInfo = function (callback) {
        socketService.emit(routeProvider.SECOND_VILLAGE_GET_INFO, {}, function (data) {
            let model = new SecondVillageModel(data)
            modelDataService.getSelectedCharacter().setSecondVillage(model)
            callback()
        })
    }

    const updateAndAnalyse = function () {
        updateSecondVillageInfo(analyse)
    }

    const analyse = function () {
        let secondVillage = modelDataService.getSelectedCharacter().getSecondVillage()

        if (!running || !secondVillage || !secondVillage.isAvailable()) {
            return false
        }

        const current = getRunningJob(secondVillage.data.jobs)

        if (current) {
            const completed = $timeHelper.server2ClientTime(current.time_completed)
            const nextRun = completed - Date.now() + 1000

            setTimeout(updateAndAnalyse, nextRun)

            return false
        }

        const collectible = getCollectibleJob(secondVillage.data.jobs)

        if (collectible) {
            return finalizeJob(collectible)
        }

        const currentDayJobs = secondVillageService.getCurrentDayJobs(secondVillage.data.jobs, secondVillage.data.day)
        const collectedJobs = secondVillageService.getCollectedJobs(secondVillage.data.jobs)
        const resources = modelDataService.getSelectedVillage().getResources().getResources()
        const availableJobs = secondVillageService.getAvailableJobs(currentDayJobs, collectedJobs, resources, [])

        if (availableJobs) {
            const firstJob = getFirstJob(availableJobs)

            startJob(firstJob, function () {
                const job = availableJobs[firstJob]
                setTimeout(updateAndAnalyse, (job.duration * 1000) + 1000)
            })
        }
    }

    let secondVillageCollector = {}

    secondVillageCollector.start = function () {
        if (!initialized) {
            return false
        }

        eventQueue.trigger(eventTypeProvider.AUTO_COLLECTOR_SECONDVILLAGE_STARTED)
        running = true
        updateAndAnalyse()
    }

    secondVillageCollector.stop = function () {
        if (!initialized) {
            return false
        }

        eventQueue.trigger(eventTypeProvider.AUTO_COLLECTOR_SECONDVILLAGE_STOPPED)
        running = false
    }

    secondVillageCollector.isRunning = function () {
        return running
    }

    secondVillageCollector.isInitialized = function () {
        return initialized
    }

    secondVillageCollector.init = function () {
        if (!secondVillageService.isFeatureActive()) {
            return false
        }

        initialized = true

        $rootScope.$on(eventTypeProvider.SECOND_VILLAGE_VILLAGE_CREATED, updateAndAnalyse)
        $rootScope.$on(eventTypeProvider.SECOND_VILLAGE_JOB_COLLECTED, updateAndAnalyse)
        $rootScope.$on(eventTypeProvider.SECOND_VILLAGE_VILLAGE_CREATED, updateAndAnalyse)
    }

    autoCollector.secondVillage = secondVillageCollector
})

require([
    'two/language',
    'two/ready',
    'two/autoCollector',
    'two/autoCollector/ui',
    'Lockr',
    'queues/EventQueue',
    'two/autoCollector/secondVillage',
    'two/autoCollector/events'
], function (
    twoLanguage,
    ready,
    autoCollector,
    autoCollectorInterface,
    Lockr,
    eventQueue
) {
    const STORAGE_KEYS = {
        ACTIVE: 'auto_collector_active'
    }

    if (autoCollector.isInitialized()) {
        return false
    }

    ready(function () {
        twoLanguage.add('auto_collector', {"de_de":{"auto_collector":{"description":"Automatischer Ressourcensammler / Ausbau des zweiten Dorfs.","activated":"Automatischer Resourcensammler aktiviert","deactivated":"Automatischer Resourcensammler deaktiviert"}},"en_us":{"auto_collector":{"description":"Automatic Resource Deposit/Second Village collector.","activated":"Automatic Collector activated","deactivated":"Automatic Collector deactivated"}},"pl_pl":{"auto_collector":{"description":"Automatyczny kolekcjoner depozytu/drugiej wioski.","activated":"Kolekcjoner aktywowany","deactivated":"Kolekcjoner deaktywowany"}},"pt_br":{"auto_collector":{"description":"Coletor autom찼tico para Dep처sito de Recursos/Segunda Aldeia.","activated":"Coletor Autom찼tico ativado","deactivated":"Coletor Autom찼tico desativado"}},"ru_ru":{"auto_collector":{"description":"�勻�棘劇逵�龜�筠�克龜橘 �閨棘� 戟逵均�逵畇 � 畇筠極棘鈞龜�逵/勻�棘�棘橘 畇筠�筠勻戟龜.","activated":"�勻�棘劇逵�龜�筠�克龜橘 �閨棘� 勻克剋��筠戟","deactivated":"�勻�棘劇逵�龜�筠�克龜橘 �閨棘� 勻�克剋��筠戟"}}})
        autoCollector.init()
        autoCollector.secondVillage.init()
        autoCollectorInterface()

        ready(function () {
            if (Lockr.get(STORAGE_KEYS.ACTIVE, false, true)) {
                autoCollector.start()
                autoCollector.secondVillage.start()
            }

            eventQueue.register(eventTypeProvider.AUTO_COLLECTOR_STARTED, function () {
                Lockr.set(STORAGE_KEYS.ACTIVE, true)
            })

            eventQueue.register(eventTypeProvider.AUTO_COLLECTOR_STOPPED, function () {
                Lockr.set(STORAGE_KEYS.ACTIVE, false)
            })
        }, ['initial_village'])
    })
})

define('two/builderQueue', [
    'two/ready',
    'two/utils',
    'two/Settings',
    'two/builderQueue/settings',
    'two/builderQueue/settings/map',
    'two/builderQueue/settings/updates',
    'two/builderQueue/types/errors',
    'conf/upgradeabilityStates',
    'conf/buildingTypes',
    'conf/locationTypes',
    'queues/EventQueue',
    'Lockr'
], function (
    ready,
    utils,
    Settings,
    SETTINGS,
    SETTINGS_MAP,
    UPDATES,
    ERROR_CODES,
    UPGRADEABILITY_STATES,
    BUILDING_TYPES,
    LOCATION_TYPES,
    eventQueue,
    Lockr
) {
    let buildingService = injector.get('buildingService')
    let premiumActionService = injector.get('premiumActionService')
    let buildingQueueService = injector.get('buildingQueueService')
    let initialized = false
    let running = false
    let localSettings
    let intervalCheckId
    let buildingSequenceLimit
    const ANALYSES_PER_MINUTE = 1
    const ANALYSES_PER_MINUTE_INSTANT_FINISH = 10
    const VILLAGE_BUILDINGS = {}
    let groupList
    let $player
    let logs
    let sequencesAvail = true
    let settings
    const STORAGE_KEYS = {
        LOGS: 'builder_queue_log',
        SETTINGS: 'builder_queue_settings'
    }

    /**
     * Loop all player villages, check if ready and init the building analyse
     * for each village.
     */
    const analyseVillages = function () {
        const villageIds = getVillageIds()

        if (!sequencesAvail) {
            builderQueue.stop()
            return false
        }

        villageIds.forEach(function (villageId) {
            const village = $player.getVillage(villageId)
            const readyState = village.checkReadyState()
            const queue = village.buildingQueue
            const jobs = queue.getAmountJobs()

            if (jobs === queue.getUnlockedSlots()) {
                return false
            }

            if (!readyState.buildingQueue || !readyState.buildings) {
                return false
            }

            analyseVillageBuildings(village)
        })
    }

    const analyseVillagesInstantFinish = function () {
        const villageIds = getVillageIds()

        villageIds.forEach(function (villageId) {
            const village = $player.getVillage(villageId)
            const queue = village.buildingQueue

            if (queue.getAmountJobs()) {
                const jobs = queue.getQueue()

                jobs.forEach(function (job) {
                    if (buildingQueueService.canBeFinishedForFree(job, village)) {
                        premiumActionService.instantBuild(job, LOCATION_TYPES.MASS_SCREEN, true, villageId)
                    }
                })
            }
        })
    }

    const initializeAllVillages = function () {
        const villageIds = getVillageIds()

        villageIds.forEach(function (villageId) {
            const village = $player.getVillage(villageId)

            if (!village.isInitialized()) {
                villageService.initializeVillage(village)
            }
        })
    }

    /**
     * Generate an Array with all player's village IDs.
     *
     * @return {Array}
     */
    const getVillageIds = function () {
        const groupVillages = settings.get(SETTINGS.GROUP_VILLAGES)
        let villages = []

        if (groupVillages) {
            villages = groupList.getGroupVillageIds(groupVillages)
            villages = villages.filter(function (vid) {
                return $player.getVillage(vid)
            })
        } else {
            angular.forEach($player.getVillages(), function (village) {
                villages.push(village.getId())
            })
        }

        return villages
    }

    /**
     * Loop all village buildings, start build job if available.
     *
     * @param {VillageModel} village
     */
    const analyseVillageBuildings = function (village) {
        let buildingLevels = angular.copy(village.buildingData.getBuildingLevels())
        const currentQueue = village.buildingQueue.getQueue()
        let sequence = angular.copy(VILLAGE_BUILDINGS)
        const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
        const activeSequenceId = settings.get(SETTINGS.ACTIVE_SEQUENCE)
        const activeSequence = sequences[activeSequenceId]

        currentQueue.forEach(function (job) {
            buildingLevels[job.building]++
        })

        if (checkVillageBuildingLimit(buildingLevels)) {
            return false
        }

        activeSequence.some(function (buildingName) {
            if (++sequence[buildingName] > buildingLevels[buildingName]) {
                buildingService.compute(village)

                checkAndUpgradeBuilding(village, buildingName, function (jobAdded, data) {
                    if (jobAdded) {
                        if (!data.job) {
                            return false
                        }

                        let now = Date.now()
                        let logData = [
                            {
                                x: village.getX(),
                                y: village.getY(),
                                name: village.getName(),
                                id: village.getId()
                            },
                            data.job.building,
                            data.job.level,
                            now
                        ]

                        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_JOB_STARTED, logData)
                        logs.unshift(logData)
                        Lockr.set(STORAGE_KEYS.LOGS, logs)
                    }
                })

                return true
            }
        })
    }

    /**
     * Init a build job
     *
     * @param {VillageModel} village
     * @param {String} buildingName - Building to be build.
     * @param {Function} callback
     */
    const checkAndUpgradeBuilding = function (village, buildingName, callback) {
        const upgradeability = checkBuildingUpgradeability(village, buildingName)

        if (upgradeability === UPGRADEABILITY_STATES.POSSIBLE) {
            upgradeBuilding(village, buildingName, function (event, data) {
                callback(true, data)
            })
        } else if (upgradeability === UPGRADEABILITY_STATES.NOT_ENOUGH_FOOD) {
            if (settings.get(SETTINGS.PRIORIZE_FARM)) {
                const limitFarm = buildingSequenceLimit[BUILDING_TYPES.FARM]
                const villageFarm = village.getBuildingData().getDataForBuilding(BUILDING_TYPES.FARM)

                if (villageFarm.level < limitFarm) {
                    upgradeBuilding(village, BUILDING_TYPES.FARM, function (event, data) {
                        callback(true, data)
                    })
                }
            }
        }

        callback(false)
    }

    const upgradeBuilding = function (village, buildingName, callback) {
        socketService.emit(routeProvider.VILLAGE_UPGRADE_BUILDING, {
            building: buildingName,
            village_id: village.getId(),
            location: LOCATION_TYPES.MASS_SCREEN,
            premium: false
        }, callback)
    }

    /**
     * Can't just use the .upgradeability value because of the preserve resources setting.
     */
    const checkBuildingUpgradeability = function (village, buildingName) {
        const buildingData = village.getBuildingData().getDataForBuilding(buildingName)

        if (buildingData.upgradeability === UPGRADEABILITY_STATES.POSSIBLE) {
            const nextLevelCosts = buildingData.nextLevelCosts
            const resources = village.getResources().getComputed()

            if (
                resources.clay.currentStock - settings.get(SETTINGS.PRESERVE_CLAY) < nextLevelCosts.clay ||
                resources.iron.currentStock - settings.get(SETTINGS.PRESERVE_IRON) < nextLevelCosts.iron ||
                resources.wood.currentStock - settings.get(SETTINGS.PRESERVE_WOOD) < nextLevelCosts.wood
            ) {
                return UPGRADEABILITY_STATES.NOT_ENOUGH_RESOURCES
            }
        }

        return buildingData.upgradeability
    }

    /**
     * Check if all buildings from the sequence already reached
     * the specified level.
     *
     * @param {Object} buildingLevels - Current buildings level from the village.
     * @return {Boolean} True if the levels already reached the limit.
     */
    const checkVillageBuildingLimit = function (buildingLevels) {
        for (let buildingName in buildingLevels) {
            if (buildingLevels[buildingName] < buildingSequenceLimit[buildingName]) {
                return false
            }
        }

        return true
    }

    /**
     * Check if the building sequence is valid by analysing if the
     * buildings exceed the maximum level.
     *
     * @param {Array} sequence
     * @return {Boolean}
     */
    const validSequence = function (sequence) {
        const buildingData = modelDataService.getGameData().getBuildings()

        for (let i = 0; i < sequence.length; i++) {
            let building = sequence[i]

            if (++sequence[building] > buildingData[building].max_level) {
                return false
            }
        }

        return true
    }

    /**
     * Get the level max for each building.
     *
     * @param {String} sequenceId
     * @return {Object} Maximum level for each building.
     */
    const getSequenceLimit = function (sequenceId) {
        const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
        const sequence = sequences[sequenceId]
        let sequenceLimit = angular.copy(VILLAGE_BUILDINGS)

        sequence.forEach(function (buildingName) {
            sequenceLimit[buildingName]++
        })

        return sequenceLimit
    }

    let builderQueue = {}

    builderQueue.start = function () {
        if (!sequencesAvail) {
            eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_NO_SEQUENCES)
            return false
        }

        running = true
        intervalCheckId = setInterval(analyseVillages, 60000 / ANALYSES_PER_MINUTE)
        intervalInstantCheckId = setInterval(analyseVillagesInstantFinish, 60000 / ANALYSES_PER_MINUTE_INSTANT_FINISH)

        ready(function () {
            initializeAllVillages()
            analyseVillages()
            analyseVillagesInstantFinish()
        }, ['all_villages_ready'])

        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_START)
    }

    builderQueue.stop = function () {
        running = false
        clearInterval(intervalCheckId)
        clearInterval(intervalInstantCheckId)
        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_STOP)
    }

    builderQueue.isRunning = function () {
        return running
    }

    builderQueue.isInitialized = function () {
        return initialized
    }

    builderQueue.getSettings = function () {
        return settings
    }

    builderQueue.getLogs = function () {
        return logs
    }

    builderQueue.clearLogs = function () {
        logs = []
        Lockr.set(STORAGE_KEYS.LOGS, logs)
        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_CLEAR_LOGS)
    }

    builderQueue.addBuildingSequence = function (id, sequence) {
        let sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)

        if (id in sequences) {
            return ERROR_CODES.SEQUENCE_EXISTS
        }

        if (!angular.isArray(sequence)) {
            return ERROR_CODES.SEQUENCE_INVALID
        }

        sequences[id] = sequence
        settings.set(SETTINGS.BUILDING_SEQUENCES, sequences, {
            quiet: true
        })
        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_ADDED, id)

        return true
    }

    builderQueue.updateBuildingSequence = function (id, sequence) {
        let sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)

        if (!(id in sequences)) {
            return ERROR_CODES.SEQUENCE_NO_EXISTS
        }

        if (!angular.isArray(sequence) || !validSequence(sequence)) {
            return ERROR_CODES.SEQUENCE_INVALID
        }

        sequences[id] = sequence
        settings.set(SETTINGS.BUILDING_SEQUENCES, sequences, {
            quiet: true
        })
        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_UPDATED, id)

        return true
    }

    builderQueue.removeSequence = function (id) {
        let sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)

        if (!(id in sequences)) {
            return ERROR_CODES.SEQUENCE_NO_EXISTS
        }

        delete sequences[id]
        settings.set(SETTINGS.BUILDING_SEQUENCES, sequences, {
            quiet: true
        })
        eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_REMOVED, id)
    }

    builderQueue.init = function () {
        initialized = true
        logs = Lockr.get(STORAGE_KEYS.LOGS, [], true)
        $player = modelDataService.getSelectedCharacter()
        groupList = modelDataService.getGroupList()

        settings = new Settings({
            settingsMap: SETTINGS_MAP,
            storageKey: STORAGE_KEYS.SETTINGS
        })

        settings.onChange(function (changes, updates, opt) {
            if (updates[UPDATES.ANALYSE]) {
                analyseVillages()
            }

            if (!opt.quiet) {
                eventQueue.trigger(eventTypeProvider.BUILDER_QUEUE_SETTINGS_CHANGE)
            }
        })

        for (let buildingName in BUILDING_TYPES) {
            VILLAGE_BUILDINGS[BUILDING_TYPES[buildingName]] = 0
        }

        sequencesAvail = Object.keys(settings.get(SETTINGS.BUILDING_SEQUENCES)).length
        buildingSequenceLimit = sequencesAvail ? getSequenceLimit(settings.get(SETTINGS.ACTIVE_SEQUENCE)) : false

        $rootScope.$on(eventTypeProvider.BUILDING_LEVEL_CHANGED, function (event, data) {
            if (!running) {
                return false
            }

            setTimeout(function () {
                let village = $player.getVillage(data.village_id)
                analyseVillageBuildings(village)
            }, 1000)
        })
    }

    return builderQueue
})

define('two/builderQueue/defaultOrders', [
    'conf/buildingTypes'
], function (
    BUILDING_TYPES
) {
    let defaultSequences = {}

    const shuffle = function (array) {
        array.sort(() => Math.random() - 0.5)
    }

    const parseSequence = function (rawSequence) {
        let parsed = []

        for (let i = 0; i < rawSequence.length; i++) {
            let item = rawSequence[i]

            if (angular.isArray(item)) {
                shuffle(item)
                parsed = parsed.concat(item)
            } else {
                parsed.push(item)
            }
        }

        return parsed
    }

    const parseSequences = function (rawSequences) {
        let parsed = {}

        for (let i in rawSequences) {
            if (rawSequences.hasOwnProperty(i)) {
                parsed[i] = parseSequence(rawSequences[i])
            }
        }

        return parsed
    }

    defaultSequences['Essential'] = [
        BUILDING_TYPES.HEADQUARTER, // 1
        BUILDING_TYPES.FARM, // 1
        BUILDING_TYPES.WAREHOUSE, // 1
        BUILDING_TYPES.RALLY_POINT, // 1
        BUILDING_TYPES.BARRACKS, // 1
        [
            // Quest: The Resources
            BUILDING_TYPES.TIMBER_CAMP, // 1
            BUILDING_TYPES.TIMBER_CAMP, // 2
            BUILDING_TYPES.CLAY_PIT, // 1
            BUILDING_TYPES.IRON_MINE, // 1

            BUILDING_TYPES.HEADQUARTER, // 2
            BUILDING_TYPES.RALLY_POINT, // 2
        ],
        [
            // Quest: First Steps
            BUILDING_TYPES.FARM, // 2
            BUILDING_TYPES.WAREHOUSE, // 2

            // Quest: Laying Down Foundation
            BUILDING_TYPES.CLAY_PIT, // 2
            BUILDING_TYPES.IRON_MINE, // 2
        ],
        [
            // Quest: More Resources
            BUILDING_TYPES.TIMBER_CAMP, // 3
            BUILDING_TYPES.CLAY_PIT, // 3
            BUILDING_TYPES.IRON_MINE, // 3

            // Quest: Resource Building
            BUILDING_TYPES.WAREHOUSE, // 3
            BUILDING_TYPES.TIMBER_CAMP, // 4
            BUILDING_TYPES.CLAY_PIT, // 4
            BUILDING_TYPES.IRON_MINE, // 4
        ],
        [
            // Quest: Get an Overview
            BUILDING_TYPES.WAREHOUSE, // 4
            BUILDING_TYPES.TIMBER_CAMP, // 5
            BUILDING_TYPES.CLAY_PIT, // 5
            BUILDING_TYPES.IRON_MINE, // 5

            // Quest: Capital
            BUILDING_TYPES.FARM, // 3
            BUILDING_TYPES.WAREHOUSE, // 5
            BUILDING_TYPES.HEADQUARTER, // 3
        ],
        [
            // Quest: The Hero
            BUILDING_TYPES.STATUE, // 1

            // Quest: Resource Expansions
            BUILDING_TYPES.TIMBER_CAMP, // 6
            BUILDING_TYPES.CLAY_PIT, // 6
            BUILDING_TYPES.IRON_MINE, // 6
        ],
        [
            // Quest: Military
            BUILDING_TYPES.BARRACKS, // 2

            // Quest: The Hospital
            BUILDING_TYPES.HEADQUARTER, // 4
            BUILDING_TYPES.TIMBER_CAMP, // 7
            BUILDING_TYPES.CLAY_PIT, // 7
            BUILDING_TYPES.IRON_MINE, // 7
            BUILDING_TYPES.FARM, // 4
            BUILDING_TYPES.HOSPITAL, // 1
        ],
        [
            // Quest: Resources
            BUILDING_TYPES.TIMBER_CAMP, // 8
            BUILDING_TYPES.CLAY_PIT, // 8
            BUILDING_TYPES.IRON_MINE, // 8
        ],
        // Quest: The Wall
        BUILDING_TYPES.WAREHOUSE, // 6
        BUILDING_TYPES.HEADQUARTER, // 5
        BUILDING_TYPES.WALL, // 1
        [
            // Quest: Village Improvements
            BUILDING_TYPES.TIMBER_CAMP, // 9
            BUILDING_TYPES.CLAY_PIT, // 9
            BUILDING_TYPES.IRON_MINE, // 9
            BUILDING_TYPES.TIMBER_CAMP, // 10
            BUILDING_TYPES.CLAY_PIT, // 10
            BUILDING_TYPES.IRON_MINE, // 10
            BUILDING_TYPES.FARM, // 5
        ],
        BUILDING_TYPES.FARM, // 6
        BUILDING_TYPES.FARM, // 7
        [
            // Quest: Hard work
            BUILDING_TYPES.TIMBER_CAMP, // 11
            BUILDING_TYPES.CLAY_PIT, // 11
            BUILDING_TYPES.IRON_MINE, // 11
            BUILDING_TYPES.TIMBER_CAMP, // 12
            BUILDING_TYPES.CLAY_PIT, // 12
            BUILDING_TYPES.IRON_MINE, // 12
        ],
        [
            // Quest: The way of defence
            BUILDING_TYPES.BARRACKS, // 3

            BUILDING_TYPES.WAREHOUSE, // 7
            BUILDING_TYPES.WAREHOUSE, // 8
            BUILDING_TYPES.FARM, // 8
            BUILDING_TYPES.WAREHOUSE, // 9
            BUILDING_TYPES.WAREHOUSE, // 10
        ],
        [
            // Quest: Market Barker
            BUILDING_TYPES.HEADQUARTER, // 6
            BUILDING_TYPES.MARKET, // 1

            // Quest: Preparations
            BUILDING_TYPES.BARRACKS, // 4
            BUILDING_TYPES.WALL, // 2
            BUILDING_TYPES.WALL, // 3
        ],
        [
            BUILDING_TYPES.FARM, // 9
            BUILDING_TYPES.FARM, // 10

            BUILDING_TYPES.BARRACKS, // 5
            BUILDING_TYPES.WAREHOUSE, // 11
            BUILDING_TYPES.FARM, // 11
        ],
        [
            BUILDING_TYPES.BARRACKS, // 6
            BUILDING_TYPES.WAREHOUSE, // 12
            BUILDING_TYPES.FARM, // 12

            BUILDING_TYPES.BARRACKS, // 7
            BUILDING_TYPES.WAREHOUSE, // 13
            BUILDING_TYPES.FARM, // 13
        ],
        [
            BUILDING_TYPES.WALL, // 4
            BUILDING_TYPES.WALL, // 5
            BUILDING_TYPES.WALL, // 6

            BUILDING_TYPES.MARKET, // 2
            BUILDING_TYPES.MARKET, // 3
            BUILDING_TYPES.MARKET, // 4
        ],
        [
            BUILDING_TYPES.BARRACKS, // 8
            BUILDING_TYPES.BARRACKS, // 9

            BUILDING_TYPES.HEADQUARTER, // 7
            BUILDING_TYPES.HEADQUARTER, // 8
        ],
        [
            BUILDING_TYPES.TAVERN, // 1
            BUILDING_TYPES.TAVERN, // 2
            BUILDING_TYPES.TAVERN, // 3

            BUILDING_TYPES.RALLY_POINT, // 3
        ],
        [
            BUILDING_TYPES.BARRACKS, // 10
            BUILDING_TYPES.BARRACKS, // 11

            BUILDING_TYPES.WAREHOUSE, // 14
            BUILDING_TYPES.FARM, // 14
        ],
        [
            BUILDING_TYPES.WAREHOUSE, // 15
            BUILDING_TYPES.FARM, // 15

            BUILDING_TYPES.BARRACKS, // 12
            BUILDING_TYPES.BARRACKS, // 13
        ],
        [
            BUILDING_TYPES.STATUE, // 2
            BUILDING_TYPES.STATUE, // 3

            BUILDING_TYPES.WALL, // 7
            BUILDING_TYPES.WALL, // 8
        ],
        [
            BUILDING_TYPES.HEADQUARTER, // 9
            BUILDING_TYPES.HEADQUARTER, // 10

            BUILDING_TYPES.WAREHOUSE, // 16
            BUILDING_TYPES.FARM, // 16
            BUILDING_TYPES.FARM, // 17
        ],
        [
            BUILDING_TYPES.IRON_MINE, // 13
            BUILDING_TYPES.IRON_MINE, // 14
            BUILDING_TYPES.IRON_MINE, // 15

            BUILDING_TYPES.WAREHOUSE, // 17
        ],
        [
            BUILDING_TYPES.BARRACKS, // 14
            BUILDING_TYPES.BARRACKS, // 15

            BUILDING_TYPES.WAREHOUSE, // 18
            BUILDING_TYPES.FARM, // 18
        ],
        [
            BUILDING_TYPES.WALL, // 9
            BUILDING_TYPES.WALL, // 10

            BUILDING_TYPES.TAVERN, // 4
            BUILDING_TYPES.TAVERN, // 5
            BUILDING_TYPES.TAVERN, // 6
        ],
        [
            BUILDING_TYPES.MARKET, // 5
            BUILDING_TYPES.MARKET, // 6
            BUILDING_TYPES.MARKET, // 7

            BUILDING_TYPES.WAREHOUSE, // 19
            BUILDING_TYPES.FARM, // 19
            BUILDING_TYPES.WAREHOUSE, // 20
            BUILDING_TYPES.FARM, // 20
            BUILDING_TYPES.WAREHOUSE, // 21
            BUILDING_TYPES.FARM, // 21
        ],
        [
            BUILDING_TYPES.IRON_MINE, // 16
            BUILDING_TYPES.IRON_MINE, // 17
            BUILDING_TYPES.IRON_MINE, // 18

            BUILDING_TYPES.RALLY_POINT, // 4
        ],
        [
            BUILDING_TYPES.BARRACKS, // 16
            BUILDING_TYPES.BARRACKS, // 17

            BUILDING_TYPES.FARM, // 22
            BUILDING_TYPES.FARM, // 23
            BUILDING_TYPES.FARM, // 24
            BUILDING_TYPES.FARM, // 25
        ],
        [
            BUILDING_TYPES.WAREHOUSE, // 22
            BUILDING_TYPES.WAREHOUSE, // 23

            BUILDING_TYPES.HEADQUARTER, // 11
            BUILDING_TYPES.HEADQUARTER, // 12
        ],
        [
            BUILDING_TYPES.STATUE, // 4
            BUILDING_TYPES.STATUE, // 5

            BUILDING_TYPES.FARM, // 26
            BUILDING_TYPES.BARRACKS, // 18
        ],
        [
            BUILDING_TYPES.HEADQUARTER, // 14
            BUILDING_TYPES.HEADQUARTER, // 15

            BUILDING_TYPES.FARM, // 27
            BUILDING_TYPES.BARRACKS, // 19
        ],
        [
            BUILDING_TYPES.HEADQUARTER, // 15
            BUILDING_TYPES.HEADQUARTER, // 16

            BUILDING_TYPES.BARRACKS, // 20

            BUILDING_TYPES.HEADQUARTER, // 17
            BUILDING_TYPES.HEADQUARTER, // 18
            BUILDING_TYPES.HEADQUARTER, // 19
            BUILDING_TYPES.HEADQUARTER, // 20
        ],
        [
            BUILDING_TYPES.ACADEMY, // 1

            BUILDING_TYPES.FARM, // 28
            BUILDING_TYPES.WAREHOUSE, // 23
            BUILDING_TYPES.WAREHOUSE, // 24
            BUILDING_TYPES.WAREHOUSE, // 25
        ],
        [
            BUILDING_TYPES.MARKET, // 8
            BUILDING_TYPES.MARKET, // 9
            BUILDING_TYPES.MARKET, // 10

            BUILDING_TYPES.TIMBER_CAMP, // 13
            BUILDING_TYPES.CLAY_PIT, // 13
            BUILDING_TYPES.IRON_MINE, // 19
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 14
            BUILDING_TYPES.CLAY_PIT, // 14
            BUILDING_TYPES.TIMBER_CAMP, // 15
            BUILDING_TYPES.CLAY_PIT, // 15

            BUILDING_TYPES.TIMBER_CAMP, // 16
            BUILDING_TYPES.TIMBER_CAMP, // 17
        ],
        [
            BUILDING_TYPES.WALL, // 11
            BUILDING_TYPES.WALL, // 12

            BUILDING_TYPES.MARKET, // 11
            BUILDING_TYPES.MARKET, // 12
            BUILDING_TYPES.MARKET, // 13
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 18
            BUILDING_TYPES.CLAY_PIT, // 16
            BUILDING_TYPES.TIMBER_CAMP, // 19
            BUILDING_TYPES.CLAY_PIT, // 17

            BUILDING_TYPES.TAVERN, // 7
            BUILDING_TYPES.TAVERN, // 8
            BUILDING_TYPES.TAVERN, // 9
        ],
        [
            BUILDING_TYPES.WALL, // 13
            BUILDING_TYPES.WALL, // 14

            BUILDING_TYPES.TIMBER_CAMP, // 20
            BUILDING_TYPES.CLAY_PIT, // 18
            BUILDING_TYPES.IRON_MINE, // 20
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 21
            BUILDING_TYPES.CLAY_PIT, // 19
            BUILDING_TYPES.IRON_MINE, // 21

            BUILDING_TYPES.BARRACKS, // 21
            BUILDING_TYPES.BARRACKS, // 22
            BUILDING_TYPES.BARRACKS, // 23
        ],
        [
            BUILDING_TYPES.FARM, // 29
            BUILDING_TYPES.WAREHOUSE, // 26
            BUILDING_TYPES.WAREHOUSE, // 27

            BUILDING_TYPES.TAVERN, // 10
            BUILDING_TYPES.TAVERN, // 11
            BUILDING_TYPES.TAVERN, // 12
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 22
            BUILDING_TYPES.CLAY_PIT, // 20
            BUILDING_TYPES.IRON_MINE, // 22

            BUILDING_TYPES.TIMBER_CAMP, // 23
            BUILDING_TYPES.CLAY_PIT, // 21
            BUILDING_TYPES.IRON_MINE, // 23
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 24
            BUILDING_TYPES.CLAY_PIT, // 22
            BUILDING_TYPES.IRON_MINE, // 24

            BUILDING_TYPES.BARRACKS, // 24
            BUILDING_TYPES.BARRACKS, // 25
        ],
        [
            BUILDING_TYPES.FARM, // 30
            BUILDING_TYPES.WAREHOUSE, // 28
            BUILDING_TYPES.WAREHOUSE, // 29

            BUILDING_TYPES.WALL, // 15
            BUILDING_TYPES.WALL, // 16
            BUILDING_TYPES.WALL, // 17
            BUILDING_TYPES.WALL, // 18
        ],
        [
            BUILDING_TYPES.TAVERN, // 13
            BUILDING_TYPES.TAVERN, // 14

            BUILDING_TYPES.RALLY_POINT, // 5

            BUILDING_TYPES.TIMBER_CAMP, // 25
            BUILDING_TYPES.CLAY_PIT, // 23
            BUILDING_TYPES.IRON_MINE, // 25
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 26
            BUILDING_TYPES.CLAY_PIT, // 24
            BUILDING_TYPES.IRON_MINE, // 26

            BUILDING_TYPES.TIMBER_CAMP, // 27
            BUILDING_TYPES.CLAY_PIT, // 25
            BUILDING_TYPES.IRON_MINE, // 27
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 28
            BUILDING_TYPES.CLAY_PIT, // 26
            BUILDING_TYPES.IRON_MINE, // 28

            BUILDING_TYPES.TIMBER_CAMP, // 29
            BUILDING_TYPES.CLAY_PIT, // 27
            BUILDING_TYPES.CLAY_PIT, // 28
            BUILDING_TYPES.IRON_MINE, // 29
        ],
        [
            BUILDING_TYPES.TIMBER_CAMP, // 30
            BUILDING_TYPES.CLAY_PIT, // 29
            BUILDING_TYPES.CLAY_PIT, // 30
            BUILDING_TYPES.IRON_MINE, // 30

            BUILDING_TYPES.WALL, // 19
            BUILDING_TYPES.WALL, // 20
        ]
    ]

    defaultSequences['Full Village'] = [
        [
            BUILDING_TYPES.HOSPITAL, // 2
            BUILDING_TYPES.HOSPITAL, // 3
            BUILDING_TYPES.HOSPITAL, // 4
            BUILDING_TYPES.HOSPITAL, // 5

            BUILDING_TYPES.MARKET, // 14
            BUILDING_TYPES.MARKET, // 15
            BUILDING_TYPES.MARKET, // 16
            BUILDING_TYPES.MARKET, // 17
        ],
        [
            BUILDING_TYPES.HEADQUARTER, // 21
            BUILDING_TYPES.HEADQUARTER, // 22
            BUILDING_TYPES.HEADQUARTER, // 23
            BUILDING_TYPES.HEADQUARTER, // 24
            BUILDING_TYPES.HEADQUARTER, // 25

            BUILDING_TYPES.PRECEPTORY, // 1

            BUILDING_TYPES.HOSPITAL, // 6
            BUILDING_TYPES.HOSPITAL, // 7
            BUILDING_TYPES.HOSPITAL, // 8
            BUILDING_TYPES.HOSPITAL, // 9
            BUILDING_TYPES.HOSPITAL, // 10
        ],
        [
            BUILDING_TYPES.MARKET, // 18
            BUILDING_TYPES.MARKET, // 19
            BUILDING_TYPES.MARKET, // 20
            BUILDING_TYPES.MARKET, // 21

            BUILDING_TYPES.PRECEPTORY, // 2
            BUILDING_TYPES.PRECEPTORY, // 3

            BUILDING_TYPES.MARKET, // 22
            BUILDING_TYPES.MARKET, // 23
            BUILDING_TYPES.MARKET, // 24
            BUILDING_TYPES.MARKET, // 25
        ],
        [
            BUILDING_TYPES.HEADQUARTER, // 26
            BUILDING_TYPES.HEADQUARTER, // 27
            BUILDING_TYPES.HEADQUARTER, // 28
            BUILDING_TYPES.HEADQUARTER, // 29
            BUILDING_TYPES.HEADQUARTER, // 30

            BUILDING_TYPES.PRECEPTORY, // 4
            BUILDING_TYPES.PRECEPTORY, // 5
            BUILDING_TYPES.PRECEPTORY, // 6
            BUILDING_TYPES.PRECEPTORY, // 7
            BUILDING_TYPES.PRECEPTORY, // 8
            BUILDING_TYPES.PRECEPTORY, // 9
            BUILDING_TYPES.PRECEPTORY, // 10
        ]
    ]

    Array.prototype.unshift.apply(
        defaultSequences['Full Village'],
        defaultSequences['Essential']
    )

    defaultSequences['Essential Without Wall'] =
        defaultSequences['Essential'].filter(function (building) {
            return building !== BUILDING_TYPES.WALL
        })

    defaultSequences['Full Wall'] = [
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL,
        BUILDING_TYPES.WALL // 20
    ]

    defaultSequences['Full Farm'] = [
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM,
        BUILDING_TYPES.FARM // 30
    ]

    return parseSequences(defaultSequences)
})

define('two/builderQueue/events', [], function () {
    angular.extend(eventTypeProvider, {
        BUILDER_QUEUE_JOB_STARTED: 'builder_queue_job_started',
        BUILDER_QUEUE_START: 'builder_queue_start',
        BUILDER_QUEUE_STOP: 'builder_queue_stop',
        BUILDER_QUEUE_UNKNOWN_SETTING: 'builder_queue_settings_unknown_setting',
        BUILDER_QUEUE_CLEAR_LOGS: 'builder_queue_clear_logs',
        BUILDER_QUEUE_BUILDING_SEQUENCES_UPDATED: 'builder_queue_building_orders_updated',
        BUILDER_QUEUE_BUILDING_SEQUENCES_ADDED: 'builder_queue_building_orders_added',
        BUILDER_QUEUE_BUILDING_SEQUENCES_REMOVED: 'builder_queue_building_orders_removed',
        BUILDER_QUEUE_SETTINGS_CHANGE: 'builder_queue_settings_change',
        BUILDER_QUEUE_NO_SEQUENCES: 'builder_queue_no_sequences'
    })
})


define('two/builderQueue/ui', [
    'two/ui',
    'two/builderQueue',
    'two/utils',
    'two/ready',
    'two/Settings',
    'two/builderQueue/settings',
    'two/builderQueue/settings/map',
    'two/builderQueue/types/errors',
    'conf/buildingTypes',
    'two/EventScope',
    'queues/EventQueue',
    'helper/time'
], function (
    interfaceOverflow,
    builderQueue,
    utils,
    ready,
    Settings,
    SETTINGS,
    SETTINGS_MAP,
    ERROR_CODES,
    BUILDING_TYPES,
    EventScope,
    eventQueue,
    timeHelper
) {
    let $scope
    let $button
    let groupList = modelDataService.getGroupList()
    let groups = []
    let buildingsLevelPoints = {}
    let running = false
    let gameDataBuildings
    let editorView = {
        sequencesAvail: true,
        modal: {}
    }
    let settings
    let settingsView = {
        sequencesAvail: true
    }
    let logsView = {}
    const TAB_TYPES = {
        SETTINGS: 'settings',
        SEQUENCES: 'sequences',
        LOGS: 'logs'
    }

    const buildingLevelReached = function (building, level) {
        const buildingData = modelDataService.getSelectedVillage().getBuildingData()
        return buildingData.getBuildingLevel(building) >= level
    }

    const buildingLevelProgress = function (building, level) {
        const queue = modelDataService.getSelectedVillage().getBuildingQueue().getQueue()
        let progress = false

        queue.some(function (job) {
            if (job.building === building && job.level === level) {
                return progress = true
            }
        })

        return progress
    }

    /**
     * Calculate the total of points accumulated ultil the specified level.
     */
    const getLevelScale = function (factor, base, level) {
        return level ? parseInt(Math.round(factor * Math.pow(base, level - 1)), 10) : 0
    }

    const moveArrayItem = function (obj, oldIndex, newIndex) {
        if (newIndex >= obj.length) {
            let i = newIndex - obj.length + 1

            while (i--) {
                obj.push(undefined)
            }
        }

        obj.splice(newIndex, 0, obj.splice(oldIndex, 1)[0])
    }

    const parseBuildingSequence = function (sequence) {
        return sequence.map(function (item) {
            return item.building
        })
    }

    const createBuildingSequence = function (sequenceId, sequence) {
        const error = builderQueue.addBuildingSequence(sequenceId, sequence)

        switch (error) {
        case ERROR_CODES.SEQUENCE_EXISTS:
            utils.emitNotif('error', $filter('i18n')('error_sequence_exists', $rootScope.loc.ale, 'builder_queue'))
            return false

            break
        case ERROR_CODES.SEQUENCE_INVALID:
            utils.emitNotif('error', $filter('i18n')('error_sequence_invalid', $rootScope.loc.ale, 'builder_queue'))
            return false

            break
        }

        return true
    }

    const selectSome = function (obj) {
        for (let i in obj) {
            if (obj.hasOwnProperty(i)) {
                return i
            }
        }

        return false
    }

    settingsView.generateSequences = function () {
        const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
        const sequencesAvail = Object.keys(sequences).length

        settingsView.sequencesAvail = sequencesAvail

        if (!sequencesAvail) {
            return false
        }

        settingsView.generateBuildingSequence()
        settingsView.generateBuildingSequenceFinal()
        settingsView.updateVisibleBuildingSequence()
    }

    settingsView.generateBuildingSequence = function () {
        const sequenceId = $scope.settings[SETTINGS.ACTIVE_SEQUENCE].value
        const buildingSequenceRaw = $scope.settings[SETTINGS.BUILDING_SEQUENCES][sequenceId]
        const buildingData = modelDataService.getGameData().getBuildings()
        let buildingLevels = {}

        settingsView.sequencesAvail = !!buildingSequenceRaw

        if (!settingsView.sequencesAvail) {
            return false
        }

        for (let building in BUILDING_TYPES) {
            buildingLevels[BUILDING_TYPES[building]] = 0
        }

        settingsView.buildingSequence = buildingSequenceRaw.map(function (building) {
            let level = ++buildingLevels[building]
            let price = buildingData[building].individual_level_costs[level]
            let state = 'not-reached'

            if (buildingLevelReached(building, level)) {
                state = 'reached'
            } else if (buildingLevelProgress(building, level)) {
                state = 'progress'
            }

            return {
                level: level,
                price: buildingData[building].individual_level_costs[level],
                building: building,
                duration: timeHelper.readableSeconds(price.build_time),
                levelPoints: buildingsLevelPoints[building][level - 1],
                state: state
            }
        })
    }

    settingsView.generateBuildingSequenceFinal = function (_sequenceId) {
        const selectedSequence = $scope.settings[SETTINGS.ACTIVE_SEQUENCE].value
        const sequenceBuildings = $scope.settings[SETTINGS.BUILDING_SEQUENCES][_sequenceId || selectedSequence]
        let sequenceObj = {}
        let sequence = []

        for (let building in gameDataBuildings) {
            sequenceObj[building] = {
                level: 0,
                order: gameDataBuildings[building].order,
                resources: {
                    wood: 0,
                    clay: 0,
                    iron: 0,
                    food: 0
                },
                points: 0,
                build_time: 0
            }
        }

        sequenceBuildings.forEach(function (building) {
            let level = ++sequenceObj[building].level
            let costs = gameDataBuildings[building].individual_level_costs[level]

            sequenceObj[building].resources.wood += parseInt(costs.wood, 10)
            sequenceObj[building].resources.clay += parseInt(costs.clay, 10)
            sequenceObj[building].resources.iron += parseInt(costs.iron, 10)
            sequenceObj[building].resources.food += parseInt(costs.food, 10)
            sequenceObj[building].build_time += parseInt(costs.build_time, 10)
            sequenceObj[building].points += buildingsLevelPoints[building][level - 1]
        })

        for (let building in sequenceObj) {
            if (sequenceObj[building].level !== 0) {
                sequence.push({
                    building: building,
                    level: sequenceObj[building].level,
                    order: sequenceObj[building].order,
                    resources: sequenceObj[building].resources,
                    points: sequenceObj[building].points,
                    build_time: sequenceObj[building].build_time
                })
            }
        }

        settingsView.buildingSequenceFinal = sequence
    }

    settingsView.updateVisibleBuildingSequence = function () {
        const offset = $scope.pagination.buildingSequence.offset
        const limit = $scope.pagination.buildingSequence.limit

        settingsView.visibleBuildingSequence = settingsView.buildingSequence.slice(offset, offset + limit)
        $scope.pagination.buildingSequence.count = settingsView.buildingSequence.length
    }

    settingsView.generateBuildingsLevelPoints = function () {
        const $gameData = modelDataService.getGameData()
        let buildingTotalPoints

        for(let buildingName in $gameData.data.buildings) {
            let buildingData = $gameData.getBuildingDataForBuilding(buildingName)
            buildingTotalPoints = 0
            buildingsLevelPoints[buildingName] = []

            for (let level = 1; level <= buildingData.max_level; level++) {
                let currentLevelPoints  = getLevelScale(buildingData.points, buildingData.points_factor, level)
                let levelPoints = currentLevelPoints - buildingTotalPoints
                buildingTotalPoints += levelPoints

                buildingsLevelPoints[buildingName].push(levelPoints)
            }
        }
    }

    editorView.moveUp = function () {
        let copy = angular.copy(editorView.buildingSequence)

        for (let i = 0; i < copy.length; i++) {
            let item = copy[i]

            if (!item.checked) {
                continue
            }

            if (i === 0) {
                continue
            }

            if (copy[i - 1].checked) {
                continue
            }

            if (copy[i - 1].building === item.building) {
                copy[i - 1].level++
                item.level--
            }

            moveArrayItem(copy, i, i - 1)
        }

        editorView.buildingSequence = copy
        editorView.updateVisibleBuildingSequence()
    }

    editorView.moveDown = function () {
        let copy = angular.copy(editorView.buildingSequence)

        for (let i = copy.length - 1; i >= 0; i--) {
            let item = copy[i]

            if (!item.checked) {
                continue
            }

            if (i === copy.length - 1) {
                continue
            }

            if (copy[i + 1].checked) {
                continue
            }

            if (copy[i + 1].building === item.building) {
                copy[i + 1].level--
                item.level++
            }

            moveArrayItem(copy, i, i + 1)
        }

        editorView.buildingSequence = copy
        editorView.updateVisibleBuildingSequence()
    }

    editorView.addBuilding = function (building, position) {
        const index = position - 1
        let newSequence = editorView.buildingSequence.slice()

        newSequence.splice(index, 0, {
            level: null,
            building: building,
            checked: false
        })

        newSequence = editorView.updateLevels(newSequence, building)

        if (!newSequence) {
            return false
        }

        editorView.buildingSequence = newSequence
        editorView.updateVisibleBuildingSequence()

        return true
    }

    editorView.removeBuilding = function (index) {
        const building = editorView.buildingSequence[index].building

        editorView.buildingSequence.splice(index, 1)
        editorView.buildingSequence = editorView.updateLevels(editorView.buildingSequence, building)

        editorView.updateVisibleBuildingSequence()
    }

    editorView.updateLevels = function (sequence, building) {
        let buildingLevel = 0
        let modifiedSequence = []
        let limitExceeded = false

        for (let i = 0; i < sequence.length; i++) {
            let item = sequence[i]

            if (item.building === building) {
                buildingLevel++

                if (buildingLevel > gameDataBuildings[building].max_level) {
                    limitExceeded = true
                    break
                }

                modifiedSequence.push({
                    level: buildingLevel,
                    building: building,
                    checked: false
                })
            } else {
                modifiedSequence.push(item)
            }
        }

        if (limitExceeded) {
            return false
        }

        return modifiedSequence
    }

    editorView.generateBuildingSequence = function () {
        const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
        const sequencesAvail = Object.keys(sequences).length

        editorView.sequencesAvail = sequencesAvail

        if (!sequencesAvail) {
            return false
        }

        const sequenceId = editorView.selectedSequence.value
        const buildingSequenceRaw = sequences[sequenceId]
        let buildingLevels = {}

        for (let building in BUILDING_TYPES) {
            buildingLevels[BUILDING_TYPES[building]] = 0
        }

        editorView.buildingSequence = buildingSequenceRaw.map(function (building) {
            return {
                level: ++buildingLevels[building],
                building: building,
                checked: false
            }
        })

        editorView.updateVisibleBuildingSequence()
    }

    editorView.updateVisibleBuildingSequence = function () {
        const offset = $scope.pagination.buildingSequenceEditor.offset
        const limit = $scope.pagination.buildingSequenceEditor.limit

        editorView.visibleBuildingSequence = editorView.buildingSequence.slice(offset, offset + limit)
        $scope.pagination.buildingSequenceEditor.count = editorView.buildingSequence.length
    }

    editorView.updateBuildingSequence = function () {
        const selectedSequence = editorView.selectedSequence.value
        const parsedSequence = parseBuildingSequence(editorView.buildingSequence)
        const error = builderQueue.updateBuildingSequence(selectedSequence, parsedSequence)

        switch (error) {
        case ERROR_CODES.SEQUENCE_NO_EXISTS:
            utils.emitNotif('error', $filter('i18n')('error_sequence_no_exits', $rootScope.loc.ale, 'builder_queue'))

            break
        case ERROR_CODES.SEQUENCE_INVALID:
            utils.emitNotif('error', $filter('i18n')('error_sequence_invalid', $rootScope.loc.ale, 'builder_queue'))

            break
        }
    }

    editorView.modal.removeSequence = function () {
        let modalScope = $rootScope.$new()

        modalScope.title = $filter('i18n')('title', $rootScope.loc.ale, 'builder_queue_remove_sequence_modal')
        modalScope.text = $filter('i18n')('text', $rootScope.loc.ale, 'builder_queue_remove_sequence_modal')
        modalScope.submitText = $filter('i18n')('remove', $rootScope.loc.ale, 'common')
        modalScope.cancelText = $filter('i18n')('cancel', $rootScope.loc.ale, 'common')

        modalScope.submit = function () {
            modalScope.closeWindow()
            builderQueue.removeSequence(editorView.selectedSequence.value)
        }

        modalScope.cancel = function () {
            modalScope.closeWindow()
        }

        windowManagerService.getModal('modal_attention', modalScope)
    }

    editorView.modal.addBuilding = function () {
        let modalScope = $rootScope.$new()

        modalScope.buildings = []
        modalScope.position = 1
        modalScope.indexLimit = editorView.buildingSequence.length + 1
        modalScope.selectedBuilding = {
            name: $filter('i18n')(BUILDING_TYPES.HEADQUARTER, $rootScope.loc.ale, 'building_names'),
            value: BUILDING_TYPES.HEADQUARTER
        }

        for (let building in gameDataBuildings) {
            modalScope.buildings.push({
                name: $filter('i18n')(building, $rootScope.loc.ale, 'building_names'),
                value: building
            })
        }

        modalScope.add = function () {
            const building = modalScope.selectedBuilding.value
            const position = modalScope.position
            const buildingName = $filter('i18n')(building, $rootScope.loc.ale, 'building_names')
            const buildingLimit = gameDataBuildings[building].max_level

            if (editorView.addBuilding(building, position)) {
                modalScope.closeWindow()
                utils.emitNotif('success', $filter('i18n')('add_building_success', $rootScope.loc.ale, 'builder_queue', buildingName, position))
            } else {
                utils.emitNotif('error', $filter('i18n')('add_building_limit_exceeded', $rootScope.loc.ale, 'builder_queue', buildingName, buildingLimit))
            }
        }

        windowManagerService.getModal('!twoverflow_builder_queue_add_building_modal', modalScope)
    }

    editorView.modal.nameSequence = function () {
        let modalScope = $rootScope.$new()
        const selectedSequenceName = editorView.selectedSequence.name
        const selectedSequence = $scope.settings[SETTINGS.BUILDING_SEQUENCES][selectedSequenceName]

        modalScope.name = selectedSequenceName

        modalScope.submit = function () {
            if (modalScope.name.length < 3) {
                utils.emitNotif('error', $filter('i18n')('name_sequence_min_lenght', $rootScope.loc.ale, 'builder_queue'))
                return false
            }

            if (createBuildingSequence(modalScope.name, selectedSequence)) {
                modalScope.closeWindow()
            }
        }

        windowManagerService.getModal('!twoverflow_builder_queue_name_sequence_modal', modalScope)
    }

    logsView.updateVisibleLogs = function () {
        const offset = $scope.pagination.logs.offset
        const limit = $scope.pagination.logs.limit

        logsView.visibleLogs = logsView.logs.slice(offset, offset + limit)
        $scope.pagination.logs.count = logsView.logs.length
    }

    logsView.clearLogs = function () {
        builderQueue.clearLogs()
    }

    const createFirstSequence = function () {
        let modalScope = $rootScope.$new()
        const initialSequence = [BUILDING_TYPES.HEADQUARTER]

        modalScope.name = ''

        modalScope.submit = function () {
            if (modalScope.name.length < 3) {
                utils.emitNotif('error', $filter('i18n')('name_sequence_min_lenght', $rootScope.loc.ale, 'builder_queue'))
                return false
            }

            if (createBuildingSequence(modalScope.name, initialSequence)) {
                $scope.settings[SETTINGS.ACTIVE_SEQUENCE] = { name: modalScope.name, value: modalScope.name }
                $scope.settings[SETTINGS.BUILDING_SEQUENCES][modalScope.name] = initialSequence

                saveSettings()

                settingsView.selectedSequence = { name: modalScope.name, value: modalScope.name }
                editorView.selectedSequence = { name: modalScope.name, value: modalScope.name }

                settingsView.generateSequences()
                editorView.generateBuildingSequence()

                modalScope.closeWindow()
                selectTab(TAB_TYPES.SEQUENCES)
            }
        }

        windowManagerService.getModal('!twoverflow_builder_queue_name_sequence_modal', modalScope)
    }

    const selectTab = function (tabType) {
        $scope.selectedTab = tabType
    }

    const saveSettings = function () {
        settings.setAll(settings.decode($scope.settings))
    }

    const switchBuilder = function () {
        if (builderQueue.isRunning()) {
            builderQueue.stop()
        } else {
            builderQueue.start()
        }
    }

    const eventHandlers = {
        updateGroups: function () {
            $scope.groups = Settings.encodeList(groupList.getGroups(), {
                type: 'groups',
                disabled: true
            })
        },
        updateSequences: function () {
            const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)

            $scope.sequences = Settings.encodeList(sequences, {
                type: 'keys',
                disabled: false
            })
        },
        generateBuildingSequences: function () {
            settingsView.generateSequences()
        },
        generateBuildingSequencesEditor: function () {
            editorView.generateBuildingSequence()
        },
        updateLogs: function () {
            $scope.logs = builderQueue.getLogs()
            logsView.updateVisibleLogs()
        },
        clearLogs: function () {
            utils.emitNotif('success', $filter('i18n')('logs_cleared', $rootScope.loc.ale, 'builder_queue'))
            eventHandlers.updateLogs()
        },
        buildingSequenceUpdate: function (event, sequenceId) {
            const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
            $scope.settings[SETTINGS.BUILDING_SEQUENCES][sequenceId] = sequences[sequenceId]

            if ($scope.settings[SETTINGS.ACTIVE_SEQUENCE].value === sequenceId) {
                settingsView.generateSequences()
            }

            utils.emitNotif('success', $filter('i18n')('sequence_updated', $rootScope.loc.ale, 'builder_queue', sequenceId))
        },
        buildingSequenceAdd: function (event, sequenceId) {
            const sequences = settings.get(SETTINGS.BUILDING_SEQUENCES)
            $scope.settings[SETTINGS.BUILDING_SEQUENCES][sequenceId] = sequences[sequenceId]
            eventHandlers.updateSequences()
            utils.emitNotif('success', $filter('i18n')('sequence_created', $rootScope.loc.ale, 'builder_queue', sequenceId))
        },
        buildingSequenceRemoved: function (event, sequenceId) {
            delete $scope.settings[SETTINGS.BUILDING_SEQUENCES][sequenceId]

            const substituteSequence = selectSome($scope.settings[SETTINGS.BUILDING_SEQUENCES])
            editorView.selectedSequence = { name: substituteSequence, value: substituteSequence }
            eventHandlers.updateSequences()
            editorView.generateBuildingSequence()

            if (settings.get(SETTINGS.ACTIVE_SEQUENCE) === sequenceId) {
                settings.set(SETTINGS.ACTIVE_SEQUENCE, substituteSequence, {
                    quiet: true
                })
                settingsView.generateSequences()
            }

            utils.emitNotif('success', $filter('i18n')('sequence_removed', $rootScope.loc.ale, 'builder_queue', sequenceId))
        },
        saveSettings: function () {
            utils.emitNotif('success', $filter('i18n')('settings_saved', $rootScope.loc.ale, 'builder_queue'))
        },
        started: function () {
            $scope.running = true
        },
        stopped: function () {
            $scope.running = false
        }
    }

    const init = function () {
        gameDataBuildings = modelDataService.getGameData().getBuildings()
        settingsView.generateBuildingsLevelPoints()
        settings = builderQueue.getSettings()

        $button = interfaceOverflow.addMenuButton('Builder', 30)
        $button.addEventListener('click', buildWindow)

        eventQueue.register(eventTypeProvider.BUILDER_QUEUE_START, function () {
            running = true
            $button.classList.remove('btn-green')
            $button.classList.add('btn-red')
            utils.emitNotif('success', $filter('i18n')('started', $rootScope.loc.ale, 'builder_queue'))
        })

        eventQueue.register(eventTypeProvider.BUILDER_QUEUE_STOP, function () {
            running = false
            $button.classList.remove('btn-red')
            $button.classList.add('btn-green')
            utils.emitNotif('success', $filter('i18n')('stopped', $rootScope.loc.ale, 'builder_queue'))
        })

        interfaceOverflow.addTemplate('twoverflow_builder_queue_window', `<div id=\"two-builder-queue\" class=\"win-content two-window\">
																																																																																																									<header class=\"win-head\">
																																																																																																										<h2>BuilderQueue</h2>
																																																																																																										<ul class=\"list-btn\">
																																																																																																											<li>
																																																																																																												<a href=\"#\" class=\"size-34x34 btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																											</ul>
																																																																																																										</header>
																																																																																																										<div class=\"win-main small-select\" scrollbar=\"\">
																																																																																																											<div class=\"tabs tabs-bg\">
																																																																																																												<div class=\"tabs-three-col\">
																																																																																																													<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.SETTINGS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.SETTINGS}\">
																																																																																																														<div class=\"tab-inner\">
																																																																																																															<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.SETTINGS}\">
																																																																																																																<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.SETTINGS}\">{{ TAB_TYPES.SETTINGS | i18n:loc.ale:'common' }}</a>
																																																																																																															</div>
																																																																																																														</div>
																																																																																																													</div>
																																																																																																													<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.SEQUENCES)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.SEQUENCES}\">
																																																																																																														<div class=\"tab-inner\">
																																																																																																															<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.SEQUENCES}\">
																																																																																																																<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.SEQUENCES}\">{{ TAB_TYPES.SEQUENCES | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																															</div>
																																																																																																														</div>
																																																																																																													</div>
																																																																																																													<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.LOGS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.LOGS}\">
																																																																																																														<div class=\"tab-inner\">
																																																																																																															<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.LOGS}\">
																																																																																																																<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.LOGS}\">{{ TAB_TYPES.LOGS | i18n:loc.ale:'common' }}</a>
																																																																																																															</div>
																																																																																																														</div>
																																																																																																													</div>
																																																																																																												</div>
																																																																																																											</div>
																																																																																																											<div class=\"box-paper footer\">
																																																																																																												<div class=\"scroll-wrap\">
																																																																																																													<div ng-show=\"selectedTab === TAB_TYPES.SETTINGS\">
																																																																																																														<h5 class=\"twx-section\">{{ 'settings' | i18n:loc.ale:'builder_queue' }}</h5>
																																																																																																														<table class=\"settings tbl-border-light tbl-striped\">
																																																																																																															<col width=\"40%\">
																																																																																																																<col>
																																																																																																																	<col width=\"60px\">
																																																																																																																		<tr>
																																																																																																																			<td>
																																																																																																																				<span class=\"ff-cell-fix\">{{ 'settings_village_groups' | i18n:loc.ale:'builder_queue' }}</span>
																																																																																																																				<td colspan=\"2\">
																																																																																																																					<div select=\"\" list=\"groups\" selected=\"settings[SETTINGS.GROUP_VILLAGES]\" drop-down=\"true\"/>
																																																																																																																					<tr ng-show=\"settingsView.sequencesAvail\">
																																																																																																																						<td>
																																																																																																																							<span class=\"ff-cell-fix\">{{ 'settings_building_sequence' | i18n:loc.ale:'builder_queue' }}</span>
																																																																																																																							<td colspan=\"2\">
																																																																																																																								<div select=\"\" list=\"sequences\" selected=\"settings[SETTINGS.ACTIVE_SEQUENCE]\" drop-down=\"true\"/>
																																																																																																																								<tr>
																																																																																																																									<td>
																																																																																																																										<span class=\"ff-cell-fix\">{{ 'settings_preserve_resources' | i18n:loc.ale:'builder_queue' }}</span>
																																																																																																																										<td colspan=\"2\">
																																																																																																																											<span class=\"icon-26x26-resource-wood\"/>
																																																																																																																											<input type=\"number\" min=\"0\" class=\"preserve-resource textfield-border text-center\" ng-model=\"settings[SETTINGS.PRESERVE_WOOD]\">
																																																																																																																												<span class=\"icon-26x26-resource-clay\"/>
																																																																																																																												<input type=\"number\" min=\"0\" class=\"preserve-resource textfield-border text-center\" ng-model=\"settings[SETTINGS.PRESERVE_CLAY]\">
																																																																																																																													<span class=\"icon-26x26-resource-iron\"/>
																																																																																																																													<input type=\"number\" min=\"0\" class=\"preserve-resource textfield-border text-center\" ng-model=\"settings[SETTINGS.PRESERVE_IRON]\">
																																																																																																																														<tr>
																																																																																																																															<td colspan=\"2\">
																																																																																																																																<span class=\"ff-cell-fix\">{{ 'settings_priorize_farm' | i18n:loc.ale:'builder_queue' }}</span>
																																																																																																																																<td>
																																																																																																																																	<div switch-slider=\"\" enabled=\"true\" border=\"true\" value=\"settings[SETTINGS.PRIORIZE_FARM]\" vertical=\"false\" size=\"'56x28'\"/>
																																																																																																																																</table>
																																																																																																																																<h5 class=\"twx-section\">{{ 'settings_building_sequence' | i18n:loc.ale:'builder_queue' }}</h5>
																																																																																																																																<p ng-show=\"!settingsView.sequencesAvail\" class=\"text-center\">
																																																																																																																																	<a href=\"#\" class=\"btn-orange btn-border create-sequence\" ng-click=\"createFirstSequence()\">{{ 'create_sequence' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																	<div ng-if=\"settingsView.sequencesAvail\">
																																																																																																																																		<div class=\"page-wrap\" pagination=\"pagination.buildingSequence\"/>
																																																																																																																																		<table class=\"tbl-border-light header-center building-sequence\">
																																																																																																																																			<col width=\"5%\">
																																																																																																																																				<col>
																																																																																																																																					<col width=\"7%\">
																																																																																																																																						<col width=\"13%\">
																																																																																																																																							<col width=\"8%\">
																																																																																																																																								<col width=\"9%\">
																																																																																																																																									<col width=\"9%\">
																																																																																																																																										<col width=\"9%\">
																																																																																																																																											<col width=\"6%\">
																																																																																																																																												<tr>
																																																																																																																																													<th tooltip=\"\" tooltip-content=\"{{ 'position' | i18n:loc.ale:'builder_queue' }}\">#<th>{{ 'building' | i18n:loc.ale:'common' }}<th>{{ 'level' | i18n:loc.ale:'common' }}<th>{{ 'duration' | i18n:loc.ale:'common' }}<th>{{ 'points' | i18n:loc.ale:'common' }}<th>
																																																																																																																																																			<span class=\"icon-26x26-resource-wood\"/>
																																																																																																																																																			<th>
																																																																																																																																																				<span class=\"icon-26x26-resource-clay\"/>
																																																																																																																																																				<th>
																																																																																																																																																					<span class=\"icon-26x26-resource-iron\"/>
																																																																																																																																																					<th>
																																																																																																																																																						<span class=\"icon-26x26-resource-food\"/>
																																																																																																																																																						<tr ng-repeat=\"item in settingsView.visibleBuildingSequence track by $index\" class=\"{{ item.state }}\">
																																																																																																																																																							<td>{{ pagination.buildingSequence.offset + $index + 1 }}<td>
																																																																																																																																																									<span class=\"building-icon icon-20x20-building-{{ item.building }}\"/> {{ item.building | i18n:loc.ale:'building_names' }}<td>{{ item.level }}<td>{{ item.duration }}<td class=\"green\">+{{ item.levelPoints | number }}<td>{{ item.price.wood | number }}<td>{{ item.price.clay | number }}<td>{{ item.price.iron | number }}<td>{{ item.price.food | number }}</table>
																																																																																																																																																															<div class=\"page-wrap\" pagination=\"pagination.buildingSequence\"/>
																																																																																																																																																														</div>
																																																																																																																																																														<h5 ng-if=\"settingsView.sequencesAvail\" class=\"twx-section\">{{ 'settings_building_sequence_final' | i18n:loc.ale:'builder_queue' }}</h5>
																																																																																																																																																														<table ng-if=\"settingsView.sequencesAvail\" class=\"tbl-border-light tbl-striped header-center building-sequence-final\">
																																																																																																																																																															<col>
																																																																																																																																																																<col width=\"5%\">
																																																																																																																																																																	<col width=\"12%\">
																																																																																																																																																																		<col width=\"8%\">
																																																																																																																																																																			<col width=\"11%\">
																																																																																																																																																																				<col width=\"11%\">
																																																																																																																																																																					<col width=\"11%\">
																																																																																																																																																																						<col width=\"7%\">
																																																																																																																																																																							<tr>
																																																																																																																																																																								<th>{{ 'building' | i18n:loc.ale:'common' }}<th>{{ 'level' | i18n:loc.ale:'common' }}<th>{{ 'duration' | i18n:loc.ale:'common' }}<th>{{ 'points' | i18n:loc.ale:'common' }}<th>
																																																																																																																																																																													<span class=\"icon-26x26-resource-wood\"/>
																																																																																																																																																																													<th>
																																																																																																																																																																														<span class=\"icon-26x26-resource-clay\"/>
																																																																																																																																																																														<th>
																																																																																																																																																																															<span class=\"icon-26x26-resource-iron\"/>
																																																																																																																																																																															<th>
																																																																																																																																																																																<span class=\"icon-26x26-resource-food\"/>
																																																																																																																																																																																<tr ng-repeat=\"item in settingsView.buildingSequenceFinal | orderBy:'order'\">
																																																																																																																																																																																	<td>
																																																																																																																																																																																		<span class=\"building-icon icon-20x20-building-{{ item.building }}\"/> {{ item.building | i18n:loc.ale:'building_names' }}<td>{{ item.level }}<td>{{ item.build_time | readableSecondsFilter }}<td class=\"green\">+{{ item.points | number }}<td>{{ item.resources.wood | number }}<td>{{ item.resources.clay | number }}<td>{{ item.resources.iron | number }}<td>{{ item.resources.food | number }}</table>
																																																																																																																																																																																							</div>
																																																																																																																																																																																							<div ng-show=\"selectedTab === TAB_TYPES.SEQUENCES\">
																																																																																																																																																																																								<h5 class=\"twx-section\">{{ 'sequences_edit_sequence' | i18n:loc.ale:'builder_queue' }}</h5>
																																																																																																																																																																																								<p ng-show=\"!editorView.sequencesAvail\" class=\"text-center\">
																																																																																																																																																																																									<a href=\"#\" class=\"btn-orange btn-border create-sequence\" ng-click=\"createFirstSequence()\">{{ 'create_sequence' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																									<table ng-if=\"editorView.sequencesAvail\" class=\"tbl-border-light tbl-striped editor-select-sequence\">
																																																																																																																																																																																										<col width=\"50%\">
																																																																																																																																																																																											<col>
																																																																																																																																																																																												<tr>
																																																																																																																																																																																													<td>
																																																																																																																																																																																														<span class=\"ff-cell-fix\">{{ 'sequences_select_edit' | i18n:loc.ale:'builder_queue' }}</span>
																																																																																																																																																																																														<td>
																																																																																																																																																																																															<div class=\"select-sequence-editor\" select=\"\" list=\"sequences\" selected=\"editorView.selectedSequence\" drop-down=\"true\"/>
																																																																																																																																																																																															<a class=\"btn btn-orange clone-sequence\" ng-click=\"editorView.modal.nameSequence()\" tooltip=\"\" tooltip-content=\"{{ 'tooltip_clone' | i18n:loc.ale:'builder_queue' }}\">{{ 'clone' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																															<a href=\"#\" class=\"btn-red remove-sequence icon-20x20-close\" ng-click=\"editorView.modal.removeSequence()\" tooltip=\"\" tooltip-content=\"{{ 'tooltip_remove_sequence' | i18n:loc.ale:'builder_queue' }}\"/>
																																																																																																																																																																																														</table>
																																																																																																																																																																																														<div ng-if=\"editorView.sequencesAvail\">
																																																																																																																																																																																															<div class=\"page-wrap\" pagination=\"pagination.buildingSequenceEditor\"/>
																																																																																																																																																																																															<table class=\"tbl-border-light tbl-striped header-center building-sequence-editor\">
																																																																																																																																																																																																<col width=\"5%\">
																																																																																																																																																																																																	<col width=\"5%\">
																																																																																																																																																																																																		<col>
																																																																																																																																																																																																			<col width=\"7%\">
																																																																																																																																																																																																				<col width=\"10%\">
																																																																																																																																																																																																					<tr>
																																																																																																																																																																																																						<th>
																																																																																																																																																																																																							<th tooltip=\"\" tooltip-content=\"{{ 'position' | i18n:loc.ale:'builder_queue' }}\">#<th>{{ 'building' | i18n:loc.ale:'common' }}<th>{{ 'level' | i18n:loc.ale:'common' }}<th>{{ 'actions' | i18n:loc.ale:'common' }}<tr ng-repeat=\"item in editorView.visibleBuildingSequence track by $index\" ng-class=\"{'selected': item.checked}\">
																																																																																																																																																																																																												<td>
																																																																																																																																																																																																													<label class=\"size-26x26 btn-orange icon-26x26-checkbox\" ng-class=\"{'icon-26x26-checkbox-checked': item.checked}\">
																																																																																																																																																																																																														<input type=\"checkbox\" ng-model=\"item.checked\">
																																																																																																																																																																																																														</label>
																																																																																																																																																																																																														<td>{{ pagination.buildingSequenceEditor.offset + $index + 1 }}<td>
																																																																																																																																																																																																																<span class=\"building-icon icon-20x20-building-{{ item.building }}\"/> {{ item.building | i18n:loc.ale:'building_names' }}<td>{{ item.level }}<td>
																																																																																																																																																																																																																		<a href=\"#\" class=\"size-20x20 btn-red icon-20x20-close\" ng-click=\"editorView.removeBuilding(pagination.buildingSequenceEditor.offset + $index)\" tooltip=\"\" tooltip-content=\"{{ 'remove_building' | i18n:loc.ale:'builder_queue' }}\"/>
																																																																																																																																																																																																																	</table>
																																																																																																																																																																																																																	<div class=\"page-wrap\" pagination=\"pagination.buildingSequenceEditor\"/>
																																																																																																																																																																																																																</div>
																																																																																																																																																																																																															</div>
																																																																																																																																																																																																															<div ng-show=\"selectedTab === TAB_TYPES.LOGS\" class=\"rich-text\">
																																																																																																																																																																																																																<div class=\"page-wrap\" pagination=\"pagination.logs\"/>
																																																																																																																																																																																																																<p class=\"text-center\" ng-show=\"!logsView.logs.length\">{{ 'logs_no_builds' | i18n:loc.ale:'builder_queue' }}<table class=\"tbl-border-light tbl-striped header-center logs\" ng-show=\"logsView.logs.length\">
																																																																																																																																																																																																																		<col width=\"40%\">
																																																																																																																																																																																																																			<col width=\"30%\">
																																																																																																																																																																																																																				<col width=\"5%\">
																																																																																																																																																																																																																					<col width=\"25%\">
																																																																																																																																																																																																																						<col>
																																																																																																																																																																																																																							<thead>
																																																																																																																																																																																																																								<tr>
																																																																																																																																																																																																																									<th>{{ 'village' | i18n:loc.ale:'common' }}<th>{{ 'building' | i18n:loc.ale:'common' }}<th>{{ 'level' | i18n:loc.ale:'common' }}<th>{{ 'started_at' | i18n:loc.ale:'common' }}<tbody>
																																																																																																																																																																																																																														<tr ng-repeat=\"log in logsView.logs\">
																																																																																																																																																																																																																															<td>
																																																																																																																																																																																																																																<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(log[0].id)\">{{ log[0].name }} ({{ log[0].x }}|{{ log[0].y }})</span>
																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																	<span class=\"building-icon icon-20x20-building-{{ log[1] }}\"/> {{ log[1] | i18n:loc.ale:'building_names' }}<td>{{ log[2] }}<td>{{ log[3] | readableDateFilter:loc.ale:GAME_TIMEZONE:GAME_TIME_OFFSET }}</table>
																																																																																																																																																																																																																																		<div class=\"page-wrap\" pagination=\"pagination.logs\"/>
																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																</div>
																																																																																																																																																																																																																															</div>
																																																																																																																																																																																																																														</div>
																																																																																																																																																																																																																														<footer class=\"win-foot\">
																																																																																																																																																																																																																															<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																<li ng-show=\"selectedTab === TAB_TYPES.SETTINGS && settingsView.sequencesAvail\">
																																																																																																																																																																																																																																	<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"saveSettings()\">{{ 'save' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																	<li ng-show=\"selectedTab === TAB_TYPES.SETTINGS && settingsView.sequencesAvail\">
																																																																																																																																																																																																																																		<a href=\"#\" ng-class=\"{false:'btn-orange', true:'btn-red'}[running]\" class=\"btn-border\" ng-click=\"switchBuilder()\">
																																																																																																																																																																																																																																			<span ng-show=\"running\">{{ 'pause' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																			<span ng-show=\"!running\">{{ 'start' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																		</a>
																																																																																																																																																																																																																																		<li ng-show=\"selectedTab === TAB_TYPES.LOGS\">
																																																																																																																																																																																																																																			<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"logsView.clearLogs()\">{{ 'logs_clear' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																																																																			<li ng-show=\"selectedTab === TAB_TYPES.SEQUENCES && editorView.sequencesAvail\">
																																																																																																																																																																																																																																				<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"editorView.moveUp()\">{{ 'sequences_move_up' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																																																																				<li ng-show=\"selectedTab === TAB_TYPES.SEQUENCES && editorView.sequencesAvail\">
																																																																																																																																																																																																																																					<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"editorView.moveDown()\">{{ 'sequences_move_down' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																																																																					<li ng-show=\"selectedTab === TAB_TYPES.SEQUENCES && editorView.sequencesAvail\">
																																																																																																																																																																																																																																						<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"editorView.modal.addBuilding()\">{{ 'sequences_add_building' | i18n:loc.ale:'builder_queue' }}</a>
																																																																																																																																																																																																																																						<li ng-show=\"selectedTab === TAB_TYPES.SEQUENCES && editorView.sequencesAvail\">
																																																																																																																																																																																																																																							<a href=\"#\" class=\"btn-border btn-red\" ng-click=\"editorView.updateBuildingSequence()\">{{ 'save' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																						</ul>
																																																																																																																																																																																																																																					</footer>
																																																																																																																																																																																																																																				</div>`)
        interfaceOverflow.addTemplate('twoverflow_builder_queue_add_building_modal', `<div id=\"add-building-modal\" class=\"win-content\">
																																																																																																																																																																																																																																					<header class=\"win-head\">
																																																																																																																																																																																																																																						<h3>{{ 'title' | i18n:loc.ale:'builder_queue_add_building_modal' }}</h3>
																																																																																																																																																																																																																																						<ul class=\"list-btn sprite\">
																																																																																																																																																																																																																																							<li>
																																																																																																																																																																																																																																								<a href=\"#\" class=\"btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																																																																																																																																																							</ul>
																																																																																																																																																																																																																																						</header>
																																																																																																																																																																																																																																						<div class=\"win-main\" scrollbar=\"\">
																																																																																																																																																																																																																																							<div class=\"box-paper\">
																																																																																																																																																																																																																																								<div class=\"scroll-wrap unit-operate-slider\">
																																																																																																																																																																																																																																									<table class=\"tbl-border-light tbl-striped header-center\">
																																																																																																																																																																																																																																										<col width=\"15%\">
																																																																																																																																																																																																																																											<col>
																																																																																																																																																																																																																																												<col width=\"15%\">
																																																																																																																																																																																																																																													<tr>
																																																																																																																																																																																																																																														<td>{{ 'building' | i18n:loc.ale:'common' }}<td colspan=\"2\">
																																																																																																																																																																																																																																																<div select=\"\" list=\"buildings\" selected=\"selectedBuilding\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																<tr>
																																																																																																																																																																																																																																																	<td>{{ 'position' | i18n:loc.ale:'builder_queue' }}<td>
																																																																																																																																																																																																																																																			<div range-slider=\"\" min=\"1\" max=\"indexLimit\" value=\"position\" enabled=\"true\"/>
																																																																																																																																																																																																																																																			<td>
																																																																																																																																																																																																																																																				<input class=\"input-border text-center\" ng-model=\"position\">
																																																																																																																																																																																																																																																				</table>
																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																																	<footer class=\"win-foot sprite-fill\">
																																																																																																																																																																																																																																																		<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																			<li>
																																																																																																																																																																																																																																																				<a href=\"#\" class=\"btn-red btn-border btn-premium\" ng-click=\"closeWindow()\">{{ 'cancel' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																				<li>
																																																																																																																																																																																																																																																					<a href=\"#\" class=\"btn-orange btn-border\" ng-click=\"add()\">{{ 'add' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																				</ul>
																																																																																																																																																																																																																																																			</footer>
																																																																																																																																																																																																																																																		</div>`)
        interfaceOverflow.addTemplate('twoverflow_builder_queue_name_sequence_modal', `<div id=\"name-sequence-modal\" class=\"win-content\">
																																																																																																																																																																																																																																																			<header class=\"win-head\">
																																																																																																																																																																																																																																																				<h3>{{ 'title' | i18n:loc.ale:'builder_queue_name_sequence_modal' }}</h3>
																																																																																																																																																																																																																																																				<ul class=\"list-btn sprite\">
																																																																																																																																																																																																																																																					<li>
																																																																																																																																																																																																																																																						<a href=\"#\" class=\"btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																																																																																																																																																																					</ul>
																																																																																																																																																																																																																																																				</header>
																																																																																																																																																																																																																																																				<div class=\"win-main\" scrollbar=\"\">
																																																																																																																																																																																																																																																					<div class=\"box-paper\">
																																																																																																																																																																																																																																																						<div class=\"scroll-wrap\">
																																																																																																																																																																																																																																																							<div class=\"box-border-light input-wrapper name_preset\">
																																																																																																																																																																																																																																																								<form ng-submit=\"submit()\">
																																																																																																																																																																																																																																																									<input focus=\"true\" ng-model=\"name\" minlength=\"3\">
																																																																																																																																																																																																																																																									</form>
																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																							</div>
																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																					<footer class=\"win-foot sprite-fill\">
																																																																																																																																																																																																																																																						<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																							<li>
																																																																																																																																																																																																																																																								<a href=\"#\" class=\"btn-red btn-border btn-premium\" ng-click=\"closeWindow()\">{{ 'cancel' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																								<li>
																																																																																																																																																																																																																																																									<a href=\"#\" class=\"btn-orange btn-border\" ng-click=\"submit()\">{{ 'add' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																								</ul>
																																																																																																																																																																																																																																																							</footer>
																																																																																																																																																																																																																																																						</div>`)
        interfaceOverflow.addStyle('#two-builder-queue tr.reached td{background-color:#b9af7e}#two-builder-queue tr.progress td{background-color:#af9d57}#two-builder-queue .building-sequence,#two-builder-queue .building-sequence-final,#two-builder-queue .building-sequence-editor,#two-builder-queue .logs{margin-bottom:10px}#two-builder-queue .building-sequence td,#two-builder-queue .building-sequence-final td,#two-builder-queue .building-sequence-editor td,#two-builder-queue .logs td,#two-builder-queue .building-sequence th,#two-builder-queue .building-sequence-final th,#two-builder-queue .building-sequence-editor th,#two-builder-queue .logs th{text-align:center}#two-builder-queue .building-sequence-editor .selected td{background-color:#b9af7e}#two-builder-queue .editor-select-sequence{margin-bottom:13px}#two-builder-queue a.btn{height:28px;line-height:28px;padding:0 10px}#two-builder-queue .clone-sequence{float:left;margin-right:10px}#two-builder-queue .select-sequence-editor{float:left;margin-top:1px;margin-right:10px}#two-builder-queue .remove-sequence{width:28px;height:28px}#two-builder-queue .create-sequence{padding:8px 20px 8px 20px}#two-builder-queue table.settings td{padding:0 5px}#two-builder-queue table.settings td:last-child{text-align:center}#two-builder-queue input.preserve-resource{width:70px}#two-builder-queue .icon-26x26-resource-wood,#two-builder-queue .icon-26x26-resource-clay,#two-builder-queue .icon-26x26-resource-iron,#two-builder-queue .icon-26x26-resource-food{transform:scale(.8);top:-1px}#add-building-modal td{text-align:center}#add-building-modal .select-wrapper{width:250px}#add-building-modal input[type="text"]{width:60px}')
    }

    const buildWindow = function () {
        const activeSequence = settings.get(SETTINGS.ACTIVE_SEQUENCE)

        $scope = $rootScope.$new()
        $scope.selectedTab = TAB_TYPES.SETTINGS
        $scope.TAB_TYPES = TAB_TYPES
        $scope.SETTINGS = SETTINGS
        $scope.running = running
        $scope.pagination = {}

        $scope.editorView = editorView
        $scope.editorView.buildingSequence = {}
        $scope.editorView.visibleBuildingSequence = {}
        $scope.editorView.selectedSequence = { name: activeSequence, value: activeSequence }

        $scope.settingsView = settingsView
        $scope.settingsView.buildingSequence = {}
        $scope.settingsView.buildingSequenceFinal = {}

        $scope.logsView = logsView
        $scope.logsView.logs = builderQueue.getLogs()

        // methods
        $scope.selectTab = selectTab
        $scope.switchBuilder = switchBuilder
        $scope.saveSettings = saveSettings
        $scope.createFirstSequence = createFirstSequence
        $scope.openVillageInfo = windowDisplayService.openVillageInfo

        settings.injectScope($scope)
        eventHandlers.updateGroups()
        eventHandlers.updateSequences()

        $scope.pagination.buildingSequence = {
            count: settingsView.buildingSequence.length,
            offset: 0,
            loader: settingsView.updateVisibleBuildingSequence,
            limit: storageService.getPaginationLimit()
        }

        $scope.pagination.buildingSequenceEditor = {
            count: editorView.buildingSequence.length,
            offset: 0,
            loader: editorView.updateVisibleBuildingSequence,
            limit: storageService.getPaginationLimit()
        }

        $scope.pagination.logs = {
            count: logsView.logs.length,
            offset: 0,
            loader: logsView.updateVisibleLogs,
            limit: storageService.getPaginationLimit()
        }

        settingsView.generateSequences()
        editorView.generateBuildingSequence()

        let eventScope = new EventScope('twoverflow_builder_queue_window')
        eventScope.register(eventTypeProvider.GROUPS_UPDATED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.GROUPS_CREATED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.GROUPS_DESTROYED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.VILLAGE_SELECTED_CHANGED, eventHandlers.generateBuildingSequences, true)
        eventScope.register(eventTypeProvider.BUILDING_UPGRADING, eventHandlers.generateBuildingSequences, true)
        eventScope.register(eventTypeProvider.BUILDING_LEVEL_CHANGED, eventHandlers.generateBuildingSequences, true)
        eventScope.register(eventTypeProvider.BUILDING_TEARING_DOWN, eventHandlers.generateBuildingSequences, true)
        eventScope.register(eventTypeProvider.VILLAGE_BUILDING_QUEUE_CHANGED, eventHandlers.generateBuildingSequences, true)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_JOB_STARTED, eventHandlers.updateLogs)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_CLEAR_LOGS, eventHandlers.clearLogs)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_UPDATED, eventHandlers.buildingSequenceUpdate)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_ADDED, eventHandlers.buildingSequenceAdd)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_BUILDING_SEQUENCES_REMOVED, eventHandlers.buildingSequenceRemoved)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_SETTINGS_CHANGE, eventHandlers.saveSettings)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_START, eventHandlers.started)
        eventScope.register(eventTypeProvider.BUILDER_QUEUE_STOP, eventHandlers.stopped)

        windowManagerService.getScreenWithInjectedScope('!twoverflow_builder_queue_window', $scope)

        $scope.$watch('settings[SETTINGS.ACTIVE_SEQUENCE].value', function (newValue, oldValue) {
            if (newValue !== oldValue) {
                eventHandlers.generateBuildingSequences()
            }
        })

        $scope.$watch('editorView.selectedSequence.value', function (newValue, oldValue) {
            if (newValue !== oldValue) {
                eventHandlers.generateBuildingSequencesEditor()
            }
        })
    }

    return init
})

define('two/builderQueue/settings', [], function () {
    return {
        GROUP_VILLAGES: 'group_villages',
        ACTIVE_SEQUENCE: 'building_sequence',
        BUILDING_SEQUENCES: 'building_orders',
        PRESERVE_WOOD: 'preserve_wood',
        PRESERVE_CLAY: 'preserve_clay',
        PRESERVE_IRON: 'preserve_iron',
        PRIORIZE_FARM: 'priorize_farm'
    }
})

define('two/builderQueue/settings/updates', [], function () {
    return {
        ANALYSE: 'analyse'
    }
})

define('two/builderQueue/settings/map', [
    'two/builderQueue/defaultOrders',
    'two/builderQueue/settings',
    'two/builderQueue/settings/updates'
], function (
    DEFAULT_ORDERS,
    SETTINGS,
    UPDATES
) {
    return {
        [SETTINGS.GROUP_VILLAGES]: {
            default: false,
            inputType: 'select',
            disabledOption: true,
            type: 'groups',
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.ACTIVE_SEQUENCE]: {
            default: 'Essential',
            inputType: 'select',
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.BUILDING_SEQUENCES]: {
            default: DEFAULT_ORDERS,
            inputType: 'buildingOrder',
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.PRESERVE_WOOD]: {
            default: 0,
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.PRESERVE_CLAY]: {
            default: 0,
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.PRESERVE_IRON]: {
            default: 0,
            updates: [UPDATES.ANALYSE]
        },
        [SETTINGS.PRIORIZE_FARM]: {
            default: false,
            inputType: 'checkbox',
            updates: [UPDATES.ANALYSE]
        }
    }
})

define('two/builderQueue/types/errors', [], function () {
    return {
        SEQUENCE_NO_EXISTS: 'sequence_no_exists',
        SEQUENCE_EXISTS: 'sequence_exists',
        SEQUENCE_INVALID: 'sequence_invalid'
    }
})

require([
    'two/language',
    'two/ready',
    'two/builderQueue',
    'two/builderQueue/ui',
    'two/builderQueue/events'
], function (
    twoLanguage,
    ready,
    builderQueue,
    builderQueueInterface
) {
    if (builderQueue.isInitialized()) {
        return false
    }

    ready(function () {
        twoLanguage.add('builder_queue', {"de_de":{"builder_queue":{"started":"Automatischer Dorfausbau gestartet","stopped":"Automatischer Dorfausbau angehalten","settings":"Einstellungen","settings_village_groups":"Ausbau von D철rfern mit den Gruppen","settings_building_sequence":"Reihenfolge","settings_building_sequence_final":"Finale Geb채udestufen","settings_priorize_farm":"Bevorzuge volle Farmen","settings_saved":"Einstellungen gespeichert!","logs_no_builds":"Kein Ausbau gestartet","logs_clear":"Protokoll l철schen","sequences":"Sequenzen","sequences_move_up":"Bewege nach oben","sequences_move_down":"Bewege nach unten","sequences_add_building":"Geb채udestufe hinzuf체gen","sequences_select_edit":"Sequenz zum 횆ndern w채hlen","sequences_edit_sequence":"Sequenz 채ndern","select_group":"Gruppe w채hlen","add_building_success":"%d wurde auf Platz %d hinzugef체gt","add_building_limit_exceeded":"%d hat die H철chststufe erreicht (%d)","position":"Platz","remove_building":"Entferne Geb채udestufe aus der Sequenz","clone":"Kopieren","tooltip_clone":"Erstelle eine neue Sequenz nach Vorlage der ausgew채hlten Sequenz","tooltip_remove_sequence":"Ausgew채hlte Sequenz l철schen","name_sequence_min_lenght":"Name muss mindestens aus 3 Zeichen bestehen.","sequence_created":"Neue Sequenz %d erstellt.","sequence_updated":"Sequenz %d aktualisiert.","sequence_removed":"Sequenz %d entfernt.","error_sequence_exists":"Diese Sequenz existiert bereits.","error_sequence_no_exists":"Diese Sequenz existiert nicht.","error_sequence_invalid":"Manche Werte dieser Sequenz sind ung체ltig.","logs_cleared":"Protokoll gel철scht.","create_sequence":"Sequenz erstellen","settings_preserve_resources":"Dorf-Ressourcen aufbewahren"},"builder_queue_add_building_modal":{"title":"Neue Geb채udestufe hinzuf체gen"},"builder_queue_name_sequence_modal":{"title":"Name der Sequenz"},"builder_queue_remove_sequence_modal":{"title":"Sequenz entfernen","text":"M철chtest Du diese Sequenz wirklich entfernen? Wenn diese Sequenz aktiv ist, wird eine andere gew채hlt und die automatische Bauschleifenbef체llung gestoppt."}},"en_us":{"builder_queue":{"started":"BuilderQueue started","stopped":"BuilderQueue stopped","settings":"Settings","settings_village_groups":"Build only on villages with the group","settings_building_sequence":"Building sequence","settings_building_sequence_final":"Buildings final levels","settings_priorize_farm":"Priorize farm if it's full","settings_saved":"Settings saved!","logs_no_builds":"No builds started","logs_clear":"Clear logs","sequences":"Sequences","sequences_move_up":"Move up","sequences_move_down":"Move down","sequences_add_building":"Add building","sequences_select_edit":"Select a sequence to edit","sequences_edit_sequence":"Edit sequence","select_group":"Select a group","add_building_success":"%d has added at position %d","add_building_limit_exceeded":"%d reached maximum level (%d)","position":"Position","remove_building":"Remove building from list","clone":"Clone","tooltip_clone":"Crate a new sequence from the selected sequence","tooltip_remove_sequence":"Remove selected sequence","name_sequence_min_lenght":"The sequence name must have at least 3 character.","sequence_created":"New sequence %d created.","sequence_updated":"Sequence %d updated.","sequence_removed":"Sequence %d removed.","error_sequence_exists":"This sequence already exists.","error_sequence_no_exists":"This sequence doesn't exist.","error_sequence_invalid":"Some sequence's value is invalid.","logs_cleared":"Logs cleared.","create_sequence":"Create a sequence","settings_preserve_resources":"Preserve village's resources"},"builder_queue_add_building_modal":{"title":"Add new building"},"builder_queue_name_sequence_modal":{"title":"Sequence name"},"builder_queue_remove_sequence_modal":{"title":"Remove sequence","text":"Are you sure to remove this sequence? If this sequence is the active one, another sequence will be selected and the BuilderQueue stopped."}},"pl_pl":{"builder_queue":{"started":"BuilderQueue Uruchomiony","stopped":"BuilderQueue Zatrzymany","settings":"Ustawienia","settings_village_groups":"Buduj w wioskach z grupy","settings_building_sequence":"Szablon kolejki budowy","settings_building_sequence_final":"Finalne poziomy budynk처w","settings_priorize_farm":"Priorize farm if it's full","settings_saved":"Ustawienia zapisane!","logs_no_builds":"Nie rozpocz휌to 탉adnej rozbudowy","logs_clear":"Wyczy힄훶 logi","sequences":"Szablony","sequences_move_up":"Przesu흦 w g처r휌","sequences_move_down":"Przesu흦 w d처흢","sequences_add_building":"Dodaj budynek","sequences_select_edit":"Wybierz szablon do edytowania","sequences_edit_sequence":"Edytuj szablon","select_group":"Wybierz grup휌","add_building_success":"%d dodany na pozycji %d","add_building_limit_exceeded":"%d osi훳gn훳흢/e흢a maksymalny poziom (%d)","position":"Pozycja","remove_building":"Usu흦 budynek z listy","clone":"Klonuj","tooltip_clone":"Utw처rz nowy szablon na podstawie wybranego szablonu","tooltip_remove_sequence":"Usu흦 wybrany szablon","name_sequence_min_lenght":"The sequence name must have at least 3 character.","sequence_created":"Nowy szablon %d utworzony.","sequence_updated":"Szablon %d zaktualizowany.","sequence_removed":"Szablon %d usuni휌ty.","error_sequence_exists":"Ten szablon ju탉 istnieje.","error_sequence_no_exists":"This sequence doesn't exist.","error_sequence_invalid":"Niekt처re z warto힄ci szablonu s훳 niepoprawne.","logs_cleared":"Logi wyczyszczone.","create_sequence":"Utw처rz szablon","settings_preserve_resources":"Preserve village's resources"},"builder_queue_add_building_modal":{"title":"Dodaj nowy budynek"},"builder_queue_name_sequence_modal":{"title":"Nazwa szablonu"},"builder_queue_remove_sequence_modal":{"title":"Usu흦 szablon","text":"Jeste힄 pewny, 탉e chcesz usun훳훶 ten szablon? Je힄li ten szablon jest teraz aktywny, inny szablon zostanie wybrany i BuilderQueue zatrzyma si휌."}},"pt_br":{"builder_queue":{"started":"BuilderQueue iniciado","stopped":"BuilderQueue parado","settings":"Configura챌천es","settings_village_groups":"Construir apenas em aldeias do grupo","settings_building_sequence":"Sequ챗ncia de constru챌천es","settings_building_sequence_final":"N챠vel final das constru챌천es","settings_priorize_farm":"Priorizar fazenda se estiver lotada","settings_saved":"Configura챌천es salvas!","logs_no_builds":"Nenhuma constru챌찾o iniciada","logs_clear":"Limpar registros","sequences":"Sequ챗ncias","sequences_move_up":"Mover acima","sequences_move_down":"Mover abaixo","sequences_add_building":"Adicionar edif챠cio","sequences_select_edit":"Selecione uma sequ챗ncia para editar","sequences_edit_sequence":"Editar sequ챗ncia","select_group":"Selecione um grupo","add_building_success":"%d foi adicionado 횪 posi챌찾o %d","add_building_limit_exceeded":"%d chegou ao n챠vel m찼ximo (%d)","position":"Posi챌찾o","remove_building":"Remover edif챠cio da lista","clone":"Clonar","tooltip_clone":"Criar uma nova sequ챗ncia a partir da sequ챗ncia selecionada","tooltip_remove_sequence":"Remover sequ챗ncia selecionada","name_sequence_min_lenght":"O nome da sequ챗ncia deve ter pelo menos 3 caracteres.","sequence_created":"Nova sequ챗ncia %d criada.","sequence_updated":"Sequ챗ncia %d atualizada.","sequence_removed":"Sequ챗ncia %d removida.","error_sequence_exists":"Esta sequ챗ncia j찼 existe.","error_sequence_no_exists":"Essa sequ챗ncia n찾o existe.","error_sequence_invalid":"Algum valor da sequ챗ncia 챕 inv찼lido.","logs_cleared":"Registro limpo.","create_sequence":"Criar uma sequ챗ncia","settings_preserve_resources":"Preservar recursos das aldeias"},"builder_queue_add_building_modal":{"title":"Adicionar novo edif챠cio"},"builder_queue_name_sequence_modal":{"title":"Nomear sequ챗ncia"},"builder_queue_remove_sequence_modal":{"title":"Remover sequ챗ncia","text":"Tem certeza que deseja remover esta sequ챗ncia? Se esta sequ챗ncia estiver ativa, outra ser찼 selecionada e o BuilderQueue ser찼 parado."}},"ru_ru":{"builder_queue":{"started":"�勻�棘劇逵�龜�筠�克棘筠 ���棘龜�筠剋���勻棘 逵克�龜勻龜�棘勻逵戟棘","stopped":"�勻�棘劇逵�龜�筠�克棘筠 ���棘龜�筠剋���勻棘 畇筠逵克�龜勻龜�棘勻逵戟棘","settings":"�逵���棘橘克龜","settings_village_groups":"鬼��棘龜�� �棘剋�克棘 勻 畇筠�筠勻戟�� � 均��極極棘橘","settings_building_sequence":"��筠�筠畇� ���棘龜�筠剋���勻逵","settings_building_sequence_final":"�逵克�龜劇逵剋�戟�橘 ��棘勻筠戟� ���棘筠戟龜�","settings_priorize_farm":"Priorize farm if it's full","settings_saved":"�逵���棘橘克龜 �棘��逵戟筠戟�!","logs_no_builds":"No builds started","logs_clear":"��龜��龜�� 菌��戟逵剋 �棘閨��龜橘","sequences":"��筠�筠畇� ���棘龜�筠剋���勻逵","sequences_move_up":"�棘畇戟��� 勻��筠","sequences_move_down":"鬼戟筠��龜","sequences_add_building":"�棘閨逵勻龜�� ���棘龜�筠剋���勻棘","sequences_select_edit":"Select a sequence to edit","sequences_edit_sequence":"�筠畇逵克�龜�棘勻逵�� 棘�筠�筠畇�","select_group":"��閨�逵�� 均��極極�","add_building_success":"%d has added at position %d","add_building_limit_exceeded":"畇棘��龜均戟�� 劇逵克�龜劇逵剋�戟�橘 ��棘勻筠戟�","position":"�棘鈞龜�龜�","remove_building":"叫畇逵剋龜�� ���棘筠戟龜筠 龜鈞 �極龜�克逵","clone":"�剋棘戟龜�棘勻逵��","tooltip_clone":"Crate a new sequence from the selected sequence","tooltip_remove_sequence":"Remove selected sequence","name_sequence_min_lenght":"The sequence name must have at least 3 character.","sequence_created":"New sequence %d created.","sequence_updated":"Sequence %d updated.","sequence_removed":"Sequence %d removed.","error_sequence_exists":"This sequence already exists.","error_sequence_no_exists":"This sequence doesn't exist.","error_sequence_invalid":"Some sequence's value is invalid.","logs_cleared":"Logs cleared.","create_sequence":"Create a sequence","settings_preserve_resources":"Preserve village's resources"},"builder_queue_add_building_modal":{"title":"�棘閨逵勻龜�� ���棘龜�筠剋���勻棘"},"builder_queue_name_sequence_modal":{"title":"Sequence name"},"builder_queue_remove_sequence_modal":{"title":"Remove sequence","text":"Are you sure to remove this sequence? If this sequence is the active one, another sequence will be selected and the BuilderQueue stopped."}}})
        builderQueue.init()
        builderQueueInterface()
    })
})

define('two/commandQueue', [
    'two/utils',
    'two/commandQueue/types/dates',
    'two/commandQueue/types/events',
    'two/commandQueue/types/filters',
    'two/commandQueue/types/commands',
    'two/commandQueue/storageKeys',
    'queues/EventQueue',
    'helper/time',
    'helper/math',
    'struct/MapData',
    'Lockr'
], function (
    utils,
    DATE_TYPES,
    EVENT_CODES,
    FILTER_TYPES,
    COMMAND_TYPES,
    STORAGE_KEYS,
    eventQueue,
    timeHelper,
    $math,
    mapData,
    Lockr
) {
    const CHECKS_PER_SECOND = 10
    const ERROR_CODES = {
        INVALID_ORIGIN: 'invalid_rigin',
        INVALID_TARGET: 'invalid_target'
    }
    let waitingCommands = []
    let waitingCommandsObject = {}
    let sentCommands = []
    let expiredCommands = []
    let running = false
    let $player
    let timeOffset

    const commandFilters = {
        [FILTER_TYPES.SELECTED_VILLAGE]: function (command) {
            return command.origin.id === modelDataService.getSelectedVillage().getId()
        },
        [FILTER_TYPES.BARBARIAN_TARGET]: function (command) {
            return !command.target.character_id
        },
        [FILTER_TYPES.ALLOWED_TYPES]: function (command, options) {
            return options[FILTER_TYPES.ALLOWED_TYPES][command.type]
        },
        [FILTER_TYPES.ATTACK]: function (command) {
            return command.type !== COMMAND_TYPES.ATTACK
        },
        [FILTER_TYPES.SUPPORT]: function (command) {
            return command.type !== COMMAND_TYPES.SUPPORT
        },
        [FILTER_TYPES.RELOCATE]: function (command) {
            return command.type !== COMMAND_TYPES.RELOCATE
        },
        [FILTER_TYPES.TEXT_MATCH]: function (command, options) {
            let show = true
            const keywords = options[FILTER_TYPES.TEXT_MATCH].toLowerCase().split(/\W/)

            const searchString = [
                command.origin.name,
                command.origin.x + '|' + command.origin.y,
                command.origin.character_name || '',
                command.target.name,
                command.target.x + '|' + command.target.y,
                command.target.character_name || '',
                command.target.tribe_name || '',
                command.target.tribe_tag || ''
            ].join('').toLowerCase()

            keywords.some(function (keyword) {
                if (keyword.length && !searchString.includes(keyword)) {
                    show = false
                    return true
                }
            })

            return show
        }
    }

    const isTimeToSend = function (sendTime) {
        return sendTime < (timeHelper.gameTime() + timeOffset)
    }

    /**
     * Remove os zeros das unidades passadas pelo jogador.
     * A raz찾o de remover 챕 por que o pr처prio n찾o os envia
     * quando os comandos s찾o enviados manualmente, ent찾o
     * caso seja enviado as unidades com valores zero poderia
     * ser uma forma de detectar os comandos autom찼ticos.
     *
     * @param  {Object} units - Unidades a serem analisadas
     * @return {Object} Objeto sem nenhum valor zero
     */
    const cleanZeroUnits = function (units) {
        let cleanUnits = {}

        for (let unit in units) {
            let amount = units[unit]

            if (amount === '*' || amount !== 0) {
                cleanUnits[unit] = amount
            }
        }

        return cleanUnits
    }

    const sortWaitingQueue = function () {
        waitingCommands = waitingCommands.sort(function (a, b) {
            return a.sendTime - b.sendTime
        })
    }

    const pushWaitingCommand = function (command) {
        waitingCommands.push(command)
    }

    const pushCommandObject = function (command) {
        waitingCommandsObject[command.id] = command
    }

    const pushSentCommand = function (command) {
        sentCommands.push(command)
    }

    const pushExpiredCommand = function (command) {
        expiredCommands.push(command)
    }

    const storeWaitingQueue = function () {
        Lockr.set(STORAGE_KEYS.QUEUE_COMMANDS, waitingCommands)
    }

    const storeSentQueue = function () {
        Lockr.set(STORAGE_KEYS.QUEUE_SENT, sentCommands)
    }

    const storeExpiredQueue = function () {
        Lockr.set(STORAGE_KEYS.QUEUE_EXPIRED, expiredCommands)
    }

    const loadStoredCommands = function () {
        const storedQueue = Lockr.get(STORAGE_KEYS.QUEUE_COMMANDS, [], true)

        if (storedQueue.length) {
            for (let i = 0; i < storedQueue.length; i++) {
                let command = storedQueue[i]

                if (timeHelper.gameTime() > command.sendTime) {
                    commandQueue.expireCommand(command, EVENT_CODES.TIME_LIMIT)
                } else {
                    waitingCommandHelpers(command)
                    pushWaitingCommand(command)
                    pushCommandObject(command)
                }
            }
        }
    }

    const waitingCommandHelpers = function (command) {
        if (command.hasOwnProperty('countdown')) {
            return false
        }

        command.countdown = function () {
            return timeHelper.readableMilliseconds((timeHelper.gameTime() + timeOffset) - command.sendTime)
        }
    }

    const parseDynamicUnits = function (command) {
        const playerVillages = modelDataService.getVillages()
        const village = playerVillages[command.origin.id]

        if (!village) {
            return EVENT_CODES.NOT_OWN_VILLAGE
        }

        const villageUnits = village.unitInfo.units
        let parsedUnits = {}

        for (let unit in command.units) {
            let amount = command.units[unit]

            if (amount === '*') {
                amount = villageUnits[unit].available

                if (amount === 0) {
                    continue
                }
            } else if (amount < 0) {
                amount = villageUnits[unit].available - Math.abs(amount)

                if (amount < 0) {
                    return EVENT_CODES.NOT_ENOUGH_UNITS
                }
            } else if (amount > 0) {
                if (amount > villageUnits[unit].available) {
                    return EVENT_CODES.NOT_ENOUGH_UNITS
                }
            }

            parsedUnits[unit] = amount
        }

        if (angular.equals({}, parsedUnits)) {
            return EVENT_CODES.NOT_ENOUGH_UNITS
        }

        return parsedUnits
    }

    const listenCommands = function () {
        setInterval(function () {
            if (!waitingCommands.length) {
                return
            }

            waitingCommands.some(function (command) {
                if (isTimeToSend(command.sendTime)) {
                    if (running) {
                        commandQueue.sendCommand(command)
                    } else {
                        commandQueue.expireCommand(command, EVENT_CODES.TIME_LIMIT)
                    }
                } else {
                    return true
                }
            })
        }, 1000 / CHECKS_PER_SECOND)
    }

    let commandQueue = {
        initialized: false
    }

    commandQueue.init = function () {
        timeOffset = utils.getTimeOffset()
        $player = modelDataService.getSelectedCharacter()

        commandQueue.initialized = true

        sentCommands = Lockr.get(STORAGE_KEYS.QUEUE_SENT, [], true)
        expiredCommands = Lockr.get(STORAGE_KEYS.QUEUE_EXPIRED, [], true)

        loadStoredCommands()
        listenCommands()

        window.addEventListener('beforeunload', function (event) {
            if (running && waitingCommands.length) {
                event.returnValue = true
            }
        })
    }

    commandQueue.sendCommand = function (command) {
        const units = parseDynamicUnits(command)

        // units === EVENT_CODES.*
        if (typeof units === 'string') {
            return commandQueue.expireCommand(command, units)
        }

        command.units = units

        socketService.emit(routeProvider.SEND_CUSTOM_ARMY, {
            start_village: command.origin.id,
            target_village: command.target.id,
            type: command.type,
            units: command.units,
            icon: 0,
            officers: command.officers,
            catapult_target: command.catapultTarget
        })

        pushSentCommand(command)
        storeSentQueue()

        commandQueue.removeCommand(command, EVENT_CODES.COMMAND_SENT)
        eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_SEND, command)
    }

    commandQueue.expireCommand = function (command, eventCode) {
        pushExpiredCommand(command)
        storeExpiredQueue()

        commandQueue.removeCommand(command, eventCode)
    }

    /**
     * UPDATE THIS SHIT BELOW
     *
     * @param {Object} command
     * @param {String} command.origin - Coordenadas da aldeia de origem.
     * @param {String} command.target - Coordenadas da aldeia alvo.
     * @param {String} command.date - Data e hora que o comando deve chegar.
     * @param {String} command.dateType - Indica se o comando vai sair ou
     *   chegar na data especificada.
     * @param {Object} command.units - Unidades que ser찾o enviados pelo comando.
     * @param {Object} command.officers - Oficiais que ser찾o enviados pelo comando.
     * @param {String} command.type - Tipo de comando.
     * @param {String=} command.catapultTarget - Alvo da catapulta, caso o comando seja um ataque.
     */
    commandQueue.addCommand = function (command) {
        if (!command.origin) {
            return eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_ORIGIN, command)
        }

        if (!command.target) {
            return eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_TARGET, command)
        }

        if (!utils.isValidDateTime(command.date)) {
            return eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_DATE, command)
        }

        if (!command.units || angular.equals(command.units, {})) {
            return eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_NO_UNITS, command)
        }

        let getOriginVillage = new Promise(function (resolve, reject) {
            commandQueue.getVillageByCoords(command.origin.x, command.origin.y, function (data) {
                data ? resolve(data) : reject(ERROR_CODES.INVALID_ORIGIN)
            })
        })

        let getTargetVillage = new Promise(function (resolve, reject) {
            commandQueue.getVillageByCoords(command.target.x, command.target.y, function (data) {
                data ? resolve(data) : reject(ERROR_CODES.INVALID_TARGET)
            })
        })

        let loadVillagesData = Promise.all([
            getOriginVillage,
            getTargetVillage
        ])

        for (let officer in command.officers) {
            if (command.officers[officer]) {
                command.officers[officer] = 1
            } else {
                delete command.officers[officer]
            }
        }

        loadVillagesData.then(function (villages) {
            command.origin = villages[0]
            command.target = villages[1]
            command.units = cleanZeroUnits(command.units)
            command.date = utils.fixDate(command.date)
            command.travelTime = utils.getTravelTime(
                command.origin,
                command.target,
                command.units,
                command.type,
                command.officers
            )

            const inputTime = utils.getTimeFromString(command.date)

            if (command.dateType === DATE_TYPES.ARRIVE) {
                command.sendTime = inputTime - command.travelTime
                command.arriveTime = inputTime
            } else if (command.dateType === DATE_TYPES.OUT) {
                command.sendTime = inputTime
                command.arriveTime = inputTime + command.travelTime
            } else {
                throw new Error('CommandQueue: wrong dateType:' + command.dateType)
            }

            if (isTimeToSend(command.sendTime)) {
                return eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_ALREADY_SENT, command)
            }

            if (command.type === COMMAND_TYPES.ATTACK && 'supporter' in command.officers) {
                delete command.officers.supporter
            }

            if (command.type === COMMAND_TYPES.ATTACK && command.units.catapult) {
                command.catapultTarget = command.catapultTarget || 'headquarter'
            } else {
                command.catapultTarget = null
            }

            command.id = utils.guid()

            waitingCommandHelpers(command)
            pushWaitingCommand(command)
            pushCommandObject(command)
            sortWaitingQueue()
            storeWaitingQueue()

            eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD, command)
        })

        loadVillagesData.catch(function (errorCode) {
            switch (errorCode) {
            case ERROR_CODES.INVALID_ORIGIN:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_ORIGIN, command)
                break
            case ERROR_CODES.INVALID_TARGET:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_TARGET, command)
                break
            }
        })
    }

    /**
     * @param  {Object} command - Dados do comando a ser removido.
     * @param {Number} eventCode - Code indicating the reason of the remotion.
     *
     * @return {Boolean} If the command was successfully removed.
     */
    commandQueue.removeCommand = function (command, eventCode) {
        let removed = false
        delete waitingCommandsObject[command.id]

        for (let i = 0; i < waitingCommands.length; i++) {
            if (waitingCommands[i].id == command.id) {
                waitingCommands.splice(i, 1)
                storeWaitingQueue()
                removed = true

                break
            }
        }

        if (removed) {
            switch (eventCode) {
            case EVENT_CODES.TIME_LIMIT:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_SEND_TIME_LIMIT, command)
                break
            case EVENT_CODES.NOT_OWN_VILLAGE:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_SEND_NOT_OWN_VILLAGE, command)
                break
            case EVENT_CODES.NOT_ENOUGH_UNITS:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_SEND_NO_UNITS_ENOUGH, command)
                break
            case EVENT_CODES.COMMAND_REMOVED:
                eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_REMOVE, command)
                break
            }

            return true
        } else {
            eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_REMOVE_ERROR, command)
            return false
        }
    }

    commandQueue.clearRegisters = function () {
        Lockr.set(STORAGE_KEYS.QUEUE_EXPIRED, [])
        Lockr.set(STORAGE_KEYS.QUEUE_SENT, [])
        expiredCommands = []
        sentCommands = []
    }

    commandQueue.start = function (disableNotif) {
        running = true
        eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_START, {
            disableNotif: !!disableNotif
        })
    }

    commandQueue.stop = function () {
        running = false
        eventQueue.trigger(eventTypeProvider.COMMAND_QUEUE_STOP)
    }

    commandQueue.isRunning = function () {
        return running
    }

    commandQueue.getWaitingCommands = function () {
        return waitingCommands
    }

    commandQueue.getWaitingCommandsObject = function () {
        return waitingCommandsObject
    }

    commandQueue.getSentCommands = function () {
        return sentCommands
    }

    commandQueue.getExpiredCommands = function () {
        return expiredCommands
    }

    commandQueue.getVillageByCoords = function (x, y, callback) {
        mapData.loadTownDataAsync(x, y, 1, 1, callback)
    }

    /**
     * @param  {String} filterId - Identifica챌찾o do filtro.
     * @param {Array=} _options - Valores a serem passados para os filtros.
     * @param {Array=} _commandsDeepFilter - Usa os comandos passados
     * pelo par창metro ao inv챕s da lista de comandos completa.
     * @return {Array} Comandos filtrados.
     */
    commandQueue.filterCommands = function (filterId, _options, _commandsDeepFilter) {
        const filterHandler = commandFilters[filterId]
        const commands = _commandsDeepFilter || waitingCommands

        return commands.filter(function (command) {
            return filterHandler(command, _options)
        })
    }

    return commandQueue
})

define('two/commandQueue/events', [], function () {
    angular.extend(eventTypeProvider, {
        COMMAND_QUEUE_SEND: 'commandqueue_send',
        COMMAND_QUEUE_SEND_TIME_LIMIT: 'commandqueue_send_time_limit',
        COMMAND_QUEUE_SEND_NOT_OWN_VILLAGE: 'commandqueue_send_not_own_village',
        COMMAND_QUEUE_SEND_NO_UNITS_ENOUGH: 'commandqueue_send_no_units_enough',
        COMMAND_QUEUE_ADD: 'commandqueue_add',
        COMMAND_QUEUE_ADD_INVALID_ORIGIN: 'commandqueue_add_invalid_origin',
        COMMAND_QUEUE_ADD_INVALID_TARGET: 'commandqueue_add_invalid_target',
        COMMAND_QUEUE_ADD_INVALID_DATE: 'commandqueue_add_invalid_date',
        COMMAND_QUEUE_ADD_NO_UNITS: 'commandqueue_add_no_units',
        COMMAND_QUEUE_ADD_ALREADY_SENT: 'commandqueue_add_already_sent',
        COMMAND_QUEUE_ADD_INVALID_ORIGIN: 'commandqueue_add_invalid_origin',
        COMMAND_QUEUE_ADD_INVALID_TARGET: 'commandqueue_add_invalid_target',
        COMMAND_QUEUE_REMOVE: 'commandqueue_remove',
        COMMAND_QUEUE_REMOVE_ERROR: 'commandqueue_remove_error',
        COMMAND_QUEUE_START: 'commandqueue_start',
        COMMAND_QUEUE_STOP: 'commandqueue_stop'
    })
})

define('two/commandQueue/ui', [
    'two/ui',
    'two/commandQueue',
    'two/EventScope',
    'two/utils',
    'two/commandQueue/types/dates',
    'two/commandQueue/types/events',
    'two/commandQueue/types/filters',
    'two/commandQueue/types/commands',
    'two/commandQueue/storageKeys',
    'queues/EventQueue',
    'helper/time',
    'helper/util',
    'Lockr'
], function (
    interfaceOverflow,
    commandQueue,
    EventScope,
    utils,
    DATE_TYPES,
    EVENT_CODES,
    FILTER_TYPES,
    COMMAND_TYPES,
    STORAGE_KEYS,
    eventQueue,
    $timeHelper,
    util,
    Lockr
) {
    let $scope
    let $button
    let $gameData = modelDataService.getGameData()
    let orderedUnitNames = $gameData.getOrderedUnitNames()
    let orderedOfficerNames = $gameData.getOrderedOfficerNames()
    let presetList = modelDataService.getPresetList()
    let mapSelectedVillage = false
    let unitOrder
    let commandData
    const TAB_TYPES = {
        ADD: 'add',
        WAITING: 'waiting',
        LOGS: 'logs'
    }
    const DEFAULT_TAB = TAB_TYPES.ADD
    const DEFAULT_CATAPULT_TARGET = 'wall'
    let attackableBuildingsList = []
    let unitList = {}
    let officerList = {}
    let timeOffset
    let activeFilters
    let filtersData
    /**
     * Name of one unity for each speed category.
     * Used to generate travel times.
     */
    const UNITS_BY_SPEED = [
        'light_cavalry',
        'heavy_cavalry',
        'archer',
        'sword',
        'ram',
        'snob',
        'trebuchet'
    ]
    const FILTER_ORDER = [
        FILTER_TYPES.SELECTED_VILLAGE,
        FILTER_TYPES.BARBARIAN_TARGET,
        FILTER_TYPES.ALLOWED_TYPES,
        FILTER_TYPES.TEXT_MATCH
    ]

    const setMapSelectedVillage = function (event, menu) {
        mapSelectedVillage = menu.data
    }

    const unsetMapSelectedVillage = function () {
        mapSelectedVillage = false
    }

    /**
     * @param {Number=} _ms - Optional time to be formated instead of the game date.
     * @return {String}
     */
    const formatedDate = function (_ms) {
        const date = new Date(_ms || ($timeHelper.gameTime() + utils.getTimeOffset()))

        const rawMS = date.getMilliseconds()
        const ms = $timeHelper.zerofill(rawMS - (rawMS % 100), 3)
        const sec = $timeHelper.zerofill(date.getSeconds(), 2)
        const min = $timeHelper.zerofill(date.getMinutes(), 2)
        const hour = $timeHelper.zerofill(date.getHours(), 2)
        const day = $timeHelper.zerofill(date.getDate(), 2)
        const month = $timeHelper.zerofill(date.getMonth() + 1, 2)
        const year = date.getFullYear()

        return hour + ':' + min + ':' + sec + ':' + ms + ' ' + day + '/' + month + '/' + year
    }

    const addDateDiff = function (date, diff) {
        if (!utils.isValidDateTime(date)) {
            return ''
        }

        date = utils.fixDate(date)
        date = utils.getTimeFromString(date)
        date += diff

        return formatedDate(date)
    }

    const updateTravelTimes = function () {
        $scope.isValidDate = utils.isValidDateTime(commandData.date)

        if (!commandData.origin || !commandData.target) {
            return false
        }

        for (let i in COMMAND_TYPES) {
            let valueType
            let date
            let commandType = COMMAND_TYPES[i]

            $scope.travelTimes[commandType] = {}

            UNITS_BY_SPEED.forEach(function (unit) {
                let travelTime = utils.getTravelTime(
                    commandData.origin,
                    commandData.target,
                    {[unit]: 1},
                    commandType,
                    commandData.officers
                )

                if ($scope.selectedDateType.value === DATE_TYPES.OUT) {
                    if ($scope.isValidDate) {
                        let date = utils.fixDate(commandData.date)
                        let outTime = utils.getTimeFromString(date)
                        valueType = isValidSendTime(outTime) ? 'valid' : 'invalid'
                    } else {
                        valueType = 'neutral'
                    }
                } else if ($scope.selectedDateType.value === DATE_TYPES.ARRIVE) {
                    if ($scope.isValidDate) {
                        let date = utils.fixDate(commandData.date)
                        let arriveTime = utils.getTimeFromString(date)
                        let sendTime = arriveTime - travelTime
                        valueType = isValidSendTime(sendTime) ? 'valid' : 'invalid'
                    } else {
                        valueType = 'invalid'
                    }
                }

                $scope.travelTimes[commandType][unit] = {
                    value: $filter('readableMillisecondsFilter')(travelTime),
                    valueType: valueType
                }
            })
        }
    }

    /**
     * @param  {Number}  time - Command date input in milliseconds.
     * @return {Boolean}
     */
    const isValidSendTime = function (time) {
        if (!$scope.isValidDate) {
            return false
        }

        return ($timeHelper.gameTime() + timeOffset) < time
    }

    const updateDateType = function () {
        commandData.dateType = $scope.selectedDateType.value
        Lockr.set(STORAGE_KEYS.LAST_DATE_TYPE, $scope.selectedDateType.value)
        updateTravelTimes()
    }

    const insertPreset = function () {
        const selectedPreset = $scope.selectedInsertPreset.value

        if (!selectedPreset) {
            return false
        }

        const presets = modelDataService.getPresetList().getPresets()
        const preset = presets[selectedPreset]

        // reset displayed value
        $scope.selectedInsertPreset = {
            name: $filter('i18n')('add_insert_preset', $rootScope.loc.ale, 'command_queue'),
            value: null
        }

        commandData.units = angular.copy(preset.units)
        commandData.officers = angular.copy(preset.officers)

        if (preset.catapult_target) {
            commandData.catapultTarget = preset.catapult_target
            $scope.catapultTarget = {
                name: $filter('i18n')(preset.catapult_target, $rootScope.loc.ale, 'building_names'),
                value: preset.catapult_target
            }
            $scope.showCatapultSelect = true

        }
    }

    const updateWaitingCommands = function () {
        $scope.waitingCommands = commandQueue.getWaitingCommands()
    }

    const updateSentCommands = function () {
        $scope.sentCommands = commandQueue.getSentCommands()
    }

    const updateExpiredCommands = function () {
        $scope.expiredCommands = commandQueue.getExpiredCommands()
    }

    const updateVisibleCommands = function () {
        let commands = $scope.waitingCommands

        FILTER_ORDER.forEach(function (filter) {
            if ($scope.activeFilters[filter]) {
                commands = commandQueue.filterCommands(filter, $scope.filtersData, commands)
            }
        })

        $scope.visibleWaitingCommands = commands
    }

    const onUnitInputFocus = function (unit) {
        if (commandData.units[unit] === 0) {
            commandData.units[unit] = ''
        }
    }

    const onUnitInputBlur = function (unit) {
        if (commandData.units[unit] === '') {
            commandData.units[unit] = 0
        }
    }

    const catapultTargetVisibility = function () {
        $scope.showCatapultSelect = !!commandData.units.catapult
    }

    const selectTab = function (tabType) {
        $scope.selectedTab = tabType
    }

    const addSelected = function () {
        const village = modelDataService.getSelectedVillage().data

        commandData.origin = {
            id: village.villageId,
            x: village.x,
            y: village.y,
            name: village.name
        }
    }

    const addMapSelected = function () {
        if (!mapSelectedVillage) {
            return utils.emitNotif('error', $filter('i18n')('error_no_map_selected_village', $rootScope.loc.ale, 'command_queue'))
        }

        commandQueue.getVillageByCoords(mapSelectedVillage.x, mapSelectedVillage.y, function (data) {
            commandData.target = {
                id: data.id,
                x: data.x,
                y: data.y,
                name: data.name
            }
        })
    }

    const addCurrentDate = function () {
        commandData.date = formatedDate()
    }

    const incrementDate = function () {
        if (!commandData.date) {
            return false
        }

        commandData.date = addDateDiff(commandData.date, 100)
    }

    const reduceDate = function () {
        if (!commandData.date) {
            return false
        }

        commandData.date = addDateDiff(commandData.date, -100)
    }

    const cleanUnitInputs = function () {
        commandData.units = angular.copy(unitList)
        commandData.officers = angular.copy(officerList)
        commandData.catapultTarget = DEFAULT_CATAPULT_TARGET
        $scope.catapultTarget = {
            name: $filter('i18n')(DEFAULT_CATAPULT_TARGET, $rootScope.loc.ale, 'building_names'),
            value: DEFAULT_CATAPULT_TARGET
        }
        $scope.showCatapultSelect = false
    }

    const addCommand = function (type) {
        let copy = angular.copy(commandData)
        copy.type = type

        commandQueue.addCommand(copy)
    }

    const clearRegisters = function () {
        commandQueue.clearRegisters()
        updateSentCommands()
        updateExpiredCommands()
    }

    const switchCommandQueue = function () {
        if (commandQueue.isRunning()) {
            commandQueue.stop()
        } else {
            commandQueue.start()
        }
    }

    /**
     * Gera um texto de notifica챌찾o com as tradu챌천es.
     *
     * @param  {String} key
     * @param  {String} key2
     * @param  {String=} prefix
     * @return {String}
     */
    const genNotifText = function (key, key2, prefix) {
        if (prefix) {
            key = prefix + '.' + key
        }

        const a = $filter('i18n')(key, $rootScope.loc.ale, 'command_queue')
        const b = $filter('i18n')(key2, $rootScope.loc.ale, 'command_queue')

        return a + ' ' + b
    }

    const toggleFilter = function (filter, allowedTypes) {
        $scope.activeFilters[filter] = !$scope.activeFilters[filter]

        if (allowedTypes) {
            $scope.filtersData[FILTER_TYPES.ALLOWED_TYPES][filter] = !$scope.filtersData[FILTER_TYPES.ALLOWED_TYPES][filter]
        }

        updateVisibleCommands()
    }

    const textMatchFilter = function () {
        $scope.activeFilters[FILTER_TYPES.TEXT_MATCH] = $scope.filtersData[FILTER_TYPES.TEXT_MATCH].length > 0
        updateVisibleCommands()
    }

    const eventHandlers = {
        updatePresets: function () {
            $scope.presets = utils.obj2selectOptions(presetList.getPresets())
        },
        autoCompleteSelected: function (event, id, data, type) {
            if (id !== 'commandqueue_village_search') {
                return false
            }

            commandData[type] = {
                id: data.raw.id,
                x: data.raw.x,
                y: data.raw.y,
                name: data.raw.name
            }

            $scope.searchQuery[type] = ''
        },
        addInvalidOrigin: function (event) {
            utils.emitNotif('error', $filter('i18n')('error_origin', $rootScope.loc.ale, 'command_queue'))
        },
        addInvalidTarget: function (event) {
            utils.emitNotif('error', $filter('i18n')('error_target', $rootScope.loc.ale, 'command_queue'))
        },
        addInvalidDate: function (event) {
            utils.emitNotif('error', $filter('i18n')('error_invalid_date', $rootScope.loc.ale, 'command_queue'))
        },
        addNoUnits: function (event) {
            utils.emitNotif('error', $filter('i18n')('error_no_units', $rootScope.loc.ale, 'command_queue'))
        },
        addAlreadySent: function (event, command) {
            const commandType = $filter('i18n')(command.type, $rootScope.loc.ale, 'common')
            const date = utils.formatDate(command.sendTime)

            utils.emitNotif('error', $filter('i18n')('error_already_sent', $rootScope.loc.ale, 'command_queue', commandType, date))
        },
        removeCommand: function (event, command) {
            updateWaitingCommands()
            updateVisibleCommands()
            $rootScope.$broadcast(eventTypeProvider.TOOLTIP_HIDE, 'twoverflow-tooltip')
            utils.emitNotif('success', genNotifText(command.type, 'removed'))
        },
        removeError: function (event, command) {
            utils.emitNotif('error', $filter('i18n')('error_remove_error', $rootScope.loc.ale, 'command_queue'))
        },
        sendTimeLimit: function (event, command) {
            updateSentCommands()
            updateExpiredCommands()
            updateWaitingCommands()
            updateVisibleCommands()
            utils.emitNotif('error', genNotifText(command.type, 'expired'))
        },
        sendNotOwnVillage: function (event, command) {
            updateSentCommands()
            updateExpiredCommands()
            updateWaitingCommands()
            updateVisibleCommands()
            utils.emitNotif('error', $filter('i18n')('error_not_own_village', $rootScope.loc.ale, 'command_queue'))
        },
        sendNoUnitsEnough: function (event, command) {
            updateSentCommands()
            updateExpiredCommands()
            updateWaitingCommands()
            updateVisibleCommands()
            utils.emitNotif('error', $filter('i18n')('error_no_units_enough', $rootScope.loc.ale, 'command_queue'))
        },
        addCommand: function (event, command) {
            updateWaitingCommands()
            updateVisibleCommands()
            utils.emitNotif('success', genNotifText(command.type, 'added'))
        },
        sendCommand: function (event, command) {
            updateSentCommands()
            updateWaitingCommands()
            updateVisibleCommands()
            utils.emitNotif('success', genNotifText(command.type, 'sent'))
        },
        start: function (event, data) {
            $scope.running = commandQueue.isRunning()

            if (data.disableNotif) {
                return false
            }

            utils.emitNotif('success', genNotifText('title', 'activated'))
        },
        stop: function (event) {
            $scope.running = commandQueue.isRunning()
            utils.emitNotif('success', genNotifText('title', 'deactivated'))
        },
        onAutoCompleteOrigin: function (data) {
            commandData.origin = {
                id: data.id,
                x: data.x,
                y: data.y,
                name: data.name
            }
        },
        onAutoCompleteTarget: function (data) {
            commandData.target = {
                id: data.id,
                x: data.x,
                y: data.y,
                name: data.name
            }
        }
    }

    const init = function () {
        timeOffset = utils.getTimeOffset()
        const attackableBuildingsMap = $gameData.getAttackableBuildings()

        for (let building in attackableBuildingsMap) {
            attackableBuildingsList.push({
                name: $filter('i18n')(building, $rootScope.loc.ale, 'building_names'),
                value: building
            })
        }

        unitOrder = angular.copy(orderedUnitNames)
        unitOrder.splice(unitOrder.indexOf('catapult'), 1)

        orderedUnitNames.forEach(function (unit) {
            unitList[unit] = 0
        })

        orderedOfficerNames.forEach(function (unit) {
            officerList[unit] = false
        })

        commandData = {
            origin: false,
            target: false,
            date: '',
            dateType: DATE_TYPES.OUT,
            units: angular.copy(unitList),
            officers: angular.copy(officerList),
            catapultTarget: DEFAULT_CATAPULT_TARGET,
            type: null
        }
        activeFilters = {
            [FILTER_TYPES.SELECTED_VILLAGE]: false,
            [FILTER_TYPES.BARBARIAN_TARGET]: false,
            [FILTER_TYPES.ALLOWED_TYPES]: true,
            [FILTER_TYPES.ATTACK]: true,
            [FILTER_TYPES.SUPPORT]: true,
            [FILTER_TYPES.RELOCATE]: true,
            [FILTER_TYPES.TEXT_MATCH]: false
        }
        filtersData = {
            [FILTER_TYPES.ALLOWED_TYPES]: {
                [FILTER_TYPES.ATTACK]: true,
                [FILTER_TYPES.SUPPORT]: true,
                [FILTER_TYPES.RELOCATE]: true,
            },
            [FILTER_TYPES.TEXT_MATCH]: ''
        }

        $button = interfaceOverflow.addMenuButton('Commander', 20)
        $button.addEventListener('click', buildWindow)

        eventQueue.register(eventTypeProvider.COMMAND_QUEUE_START, function () {
            $button.classList.remove('btn-green')
            $button.classList.add('btn-red')
        })

        eventQueue.register(eventTypeProvider.COMMAND_QUEUE_STOP, function () {
            $button.classList.remove('btn-red')
            $button.classList.add('btn-green')
        })

        $rootScope.$on(eventTypeProvider.SHOW_CONTEXT_MENU, setMapSelectedVillage)
        $rootScope.$on(eventTypeProvider.DESTROY_CONTEXT_MENU, unsetMapSelectedVillage)

        interfaceOverflow.addTemplate('twoverflow_queue_window', `<div id=\"two-command-queue\" class=\"win-content two-window\">
																																																																																																																																																																																																																																																																<header class=\"win-head\">
																																																																																																																																																																																																																																																																	<h2>CommandQueue</h2>
																																																																																																																																																																																																																																																																	<ul class=\"list-btn\">
																																																																																																																																																																																																																																																																		<li>
																																																																																																																																																																																																																																																																			<a href=\"#\" class=\"size-34x34 btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																																																																																																																																																																																		</ul>
																																																																																																																																																																																																																																																																	</header>
																																																																																																																																																																																																																																																																	<div class=\"win-main small-select\" scrollbar=\"\">
																																																																																																																																																																																																																																																																		<div class=\"tabs tabs-bg\">
																																																																																																																																																																																																																																																																			<div class=\"tabs-three-col\">
																																																																																																																																																																																																																																																																				<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.ADD)\" ng-class=\"{true:'tab-active', false:''}[selectedTab == TAB_TYPES.ADD]\">
																																																																																																																																																																																																																																																																					<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																						<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.ADD}\">
																																																																																																																																																																																																																																																																							<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.ADD}\">{{ 'tab_add' | i18n:loc.ale:'command_queue' }}</a>
																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																				<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.WAITING)\" ng-class=\"{true:'tab-active', false:''}[selectedTab == TAB_TYPES.WAITING]\">
																																																																																																																																																																																																																																																																					<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																						<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.WAITING}\">
																																																																																																																																																																																																																																																																							<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.WAITING}\">{{ 'tab_waiting' | i18n:loc.ale:'command_queue' }}</a>
																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																				<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.LOGS)\" ng-class=\"{true:'tab-active', false:''}[selectedTab == TAB_TYPES.LOGS]\">
																																																																																																																																																																																																																																																																					<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																						<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.LOGS}\">
																																																																																																																																																																																																																																																																							<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.LOGS}\">{{ 'tab_logs' | i18n:loc.ale:'command_queue' }}</a>
																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																																		<div class=\"box-paper footer\">
																																																																																																																																																																																																																																																																			<div class=\"scroll-wrap\">
																																																																																																																																																																																																																																																																				<div class=\"add\" ng-show=\"selectedTab === TAB_TYPES.ADD\">
																																																																																																																																																																																																																																																																					<form class=\"addForm\">
																																																																																																																																																																																																																																																																						<div>
																																																																																																																																																																																																																																																																							<table class=\"tbl-border-light tbl-striped basic-config\">
																																																																																																																																																																																																																																																																								<col width=\"30%\">
																																																																																																																																																																																																																																																																									<col width=\"5%\">
																																																																																																																																																																																																																																																																										<col>
																																																																																																																																																																																																																																																																											<col width=\"18%\">
																																																																																																																																																																																																																																																																												<tr>
																																																																																																																																																																																																																																																																													<td>
																																																																																																																																																																																																																																																																														<div auto-complete=\"autoCompleteOrigin\"/>
																																																																																																																																																																																																																																																																														<td class=\"text-center\">
																																																																																																																																																																																																																																																																															<span class=\"icon-26x26-rte-village\"/>
																																																																																																																																																																																																																																																																															<td ng-if=\"!commandData.origin\">{{ 'add_no_village' | i18n:loc.ale:'command_queue' }}<td ng-if=\"commandData.origin\">{{ commandData.origin.name }} ({{ commandData.origin.x }}|{{ commandData.origin.y }})<td class=\"actions\">
																																																																																																																																																																																																																																																																																		<a class=\"btn btn-orange\" ng-click=\"addSelected()\" tooltip=\"\" tooltip-content=\"{{ 'add_selected' | i18n:loc.ale:'command_queue' }}\">{{ 'selected' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																			<td>
																																																																																																																																																																																																																																																																																				<div auto-complete=\"autoCompleteTarget\"/>
																																																																																																																																																																																																																																																																																				<td class=\"text-center\">
																																																																																																																																																																																																																																																																																					<span class=\"icon-26x26-rte-village\"/>
																																																																																																																																																																																																																																																																																					<td ng-if=\"!commandData.target\">{{ 'add_no_village' | i18n:loc.ale:'command_queue' }}<td ng-if=\"commandData.target\">{{ commandData.target.name }} ({{ commandData.target.x }}|{{ commandData.target.y }})<td class=\"actions\">
																																																																																																																																																																																																																																																																																								<a class=\"btn btn-orange\" ng-click=\"addMapSelected()\" tooltip=\"\" tooltip-content=\"{{ 'add_map_selected' | i18n:loc.ale:'command_queue' }}\">{{ 'selected' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																								<tr>
																																																																																																																																																																																																																																																																																									<td>
																																																																																																																																																																																																																																																																																										<input ng-model=\"commandData.date\" class=\"textfield-border date\" pattern=\"\\s*\\d{1,2}:\\d{1,2}:\\d{1,2}(:\\d{1,3})? \\d{1,2}\\/\\d{1,2}\\/\\d{4}\\s*\" placeholder=\"{{ 'add_date' | i18n:loc.ale:'command_queue' }}\" tooltip=\"\" tooltip-content=\"hh:mm:ss:SSS dd/MM/yyyy\">
																																																																																																																																																																																																																																																																																											<td class=\"text-center\">
																																																																																																																																																																																																																																																																																												<span class=\"icon-26x26-time\"/>
																																																																																																																																																																																																																																																																																												<td>
																																																																																																																																																																																																																																																																																													<div select=\"\" list=\"dateTypes\" selected=\"selectedDateType\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																													<td class=\"actions\">
																																																																																																																																																																																																																																																																																														<a class=\"btn btn-orange\" ng-click=\"reduceDate()\" tooltip=\"\" tooltip-content=\"{{ 'add_current_date_minus' | i18n:loc.ale:'command_queue' }}\">-</a>
																																																																																																																																																																																																																																																																																														<a class=\"btn btn-orange\" ng-click=\"addCurrentDate()\" tooltip=\"\" tooltip-content=\"{{ 'add_current_date' | i18n:loc.ale:'command_queue' }}\">{{ 'now' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																														<a class=\"btn btn-orange\" ng-click=\"incrementDate()\" tooltip=\"\" tooltip-content=\"{{ 'add_current_date_plus' | i18n:loc.ale:'command_queue' }}\">+</a>
																																																																																																																																																																																																																																																																																													</table>
																																																																																																																																																																																																																																																																																													<table ng-show=\"commandData.origin && commandData.target\" class=\"tbl-border-light tbl-units tbl-speed screen-village-info\">
																																																																																																																																																																																																																																																																																														<thead>
																																																																																																																																																																																																																																																																																															<tr>
																																																																																																																																																																																																																																																																																																<th colspan=\"7\">{{ 'speed_title' | i18n:loc.ale:'screen_village_info' }}<tbody>
																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																			<td class=\"odd\">
																																																																																																																																																																																																																																																																																																				<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																					<span class=\"icon icon-34x34-unit-knight\" tooltip=\"\" tooltip-content=\"{{ 'knight' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																					<span class=\"icon icon-34x34-unit-light_cavalry\" tooltip=\"\" tooltip-content=\"{{ 'light_cavalry' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																					<span class=\"icon icon-34x34-unit-mounted_archer\" tooltip=\"\" tooltip-content=\"{{ 'mounted_archer' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																				<div>
																																																																																																																																																																																																																																																																																																					<div class=\"box-time-sub-icon time-support\" ng-class=\"{'text-red': travelTimes.support.light_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.support.light_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">
																																																																																																																																																																																																																																																																																																						<div class=\"time-icon icon-20x20-support-check\"/>{{ travelTimes.support.light_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																					<div class=\"box-time-sub-icon time-attack\" ng-class=\"{'text-red': travelTimes.attack.light_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.attack.light_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">
																																																																																																																																																																																																																																																																																																						<div class=\"time-icon icon-20x20-attack-check\"/>{{ travelTimes.attack.light_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																					<div class=\"box-time-sub-icon time-relocate\" ng-class=\"{'text-red': travelTimes.relocate.light_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.relocate.light_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">
																																																																																																																																																																																																																																																																																																						<div class=\"time-icon icon-20x20-relocate\"/>{{ travelTimes.relocate.light_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																				<td>
																																																																																																																																																																																																																																																																																																					<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																						<span class=\"icon icon-single icon-34x34-unit-heavy_cavalry\" tooltip=\"\" tooltip-content=\"{{ 'heavy_cavalry' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																					<div>
																																																																																																																																																																																																																																																																																																						<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.heavy_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.support.heavy_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.heavy_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																						<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.heavy_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.support.heavy_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.heavy_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																						<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.heavy_cavalry.valueType == 'invalid', 'text-green-bright': travelTimes.support.heavy_cavalry.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.relocate.heavy_cavalry.value }}</div>
																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																					<td class=\"odd\">
																																																																																																																																																																																																																																																																																																						<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																							<span class=\"icon icon-34x34-unit-archer\" tooltip=\"\" tooltip-content=\"{{ 'archer' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																							<span class=\"icon icon-34x34-unit-spear\" tooltip=\"\" tooltip-content=\"{{ 'spear' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																							<span class=\"icon icon-34x34-unit-axe\" tooltip=\"\" tooltip-content=\"{{ 'axe' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																							<span class=\"icon icon-34x34-unit-doppelsoldner\" tooltip=\"\" tooltip-content=\"{{ 'doppelsoldner' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																																																						<div>
																																																																																																																																																																																																																																																																																																							<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.archer.valueType == 'invalid', 'text-green-bright': travelTimes.support.archer.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.archer.value }}</div>
																																																																																																																																																																																																																																																																																																							<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.archer.valueType == 'invalid', 'text-green-bright': travelTimes.support.archer.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.archer.value }}</div>
																																																																																																																																																																																																																																																																																																							<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.archer.valueType == 'invalid', 'text-green-bright': travelTimes.support.archer.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.relocate.archer.value }}</div>
																																																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																																																						<td>
																																																																																																																																																																																																																																																																																																							<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																								<span class=\"icon icon-single icon-34x34-unit-sword\" tooltip=\"\" tooltip-content=\"{{ 'sword' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																							</div>
																																																																																																																																																																																																																																																																																																							<div>
																																																																																																																																																																																																																																																																																																								<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.sword.valueType == 'invalid', 'text-green-bright': travelTimes.support.sword.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.sword.value }}</div>
																																																																																																																																																																																																																																																																																																								<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.sword.valueType == 'invalid', 'text-green-bright': travelTimes.support.sword.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.sword.value }}</div>
																																																																																																																																																																																																																																																																																																								<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.sword.valueType == 'invalid', 'text-green-bright': travelTimes.support.sword.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.relocate.sword.value }}</div>
																																																																																																																																																																																																																																																																																																							</div>
																																																																																																																																																																																																																																																																																																							<td class=\"odd\">
																																																																																																																																																																																																																																																																																																								<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																									<span class=\"icon icon-34x34-unit-catapult\" tooltip=\"\" tooltip-content=\"{{ 'catapult' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																									<span class=\"icon icon-34x34-unit-ram\" tooltip=\"\" tooltip-content=\"{{ 'ram' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																								<div>
																																																																																																																																																																																																																																																																																																									<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.ram.valueType == 'invalid', 'text-green-bright': travelTimes.support.ram.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.ram.value }}</div>
																																																																																																																																																																																																																																																																																																									<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.ram.valueType == 'invalid', 'text-green-bright': travelTimes.support.ram.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.ram.value }}</div>
																																																																																																																																																																																																																																																																																																									<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.ram.valueType == 'invalid', 'text-green-bright': travelTimes.support.ram.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.relocate.ram.value }}</div>
																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																								<td>
																																																																																																																																																																																																																																																																																																									<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																										<span class=\"icon icon-single icon-34x34-unit-snob\" tooltip=\"\" tooltip-content=\"{{ 'snob' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																									</div>
																																																																																																																																																																																																																																																																																																									<div>
																																																																																																																																																																																																																																																																																																										<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.snob.valueType == 'invalid', 'text-green-bright': travelTimes.support.snob.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.snob.value }}</div>
																																																																																																																																																																																																																																																																																																										<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.snob.valueType == 'invalid', 'text-green-bright': travelTimes.support.snob.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.snob.value }}</div>
																																																																																																																																																																																																																																																																																																										<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.snob.valueType == 'invalid', 'text-green-bright': travelTimes.support.snob.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'cannot_relocate_snob' | i18n:loc.ale:'military_operations' }}\">-</div>
																																																																																																																																																																																																																																																																																																									</div>
																																																																																																																																																																																																																																																																																																									<td class=\"odd\">
																																																																																																																																																																																																																																																																																																										<div class=\"unit-wrap\">
																																																																																																																																																																																																																																																																																																											<span class=\"icon icon-single icon-34x34-unit-trebuchet\" tooltip=\"\" tooltip-content=\"{{ 'trebuchet' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																										</div>
																																																																																																																																																																																																																																																																																																										<div>
																																																																																																																																																																																																																																																																																																											<div class=\"box-time-sub time-support\" ng-class=\"{'text-red': travelTimes.support.trebuchet.valueType == 'invalid', 'text-green-bright': travelTimes.support.trebuchet.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_support' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.support.trebuchet.value }}</div>
																																																																																																																																																																																																																																																																																																											<div class=\"box-time-sub time-attack\" ng-class=\"{'text-red': travelTimes.support.trebuchet.valueType == 'invalid', 'text-green-bright': travelTimes.support.trebuchet.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_attack' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.attack.trebuchet.value }}</div>
																																																																																																																																																																																																																																																																																																											<div class=\"box-time-sub time-relocate\" ng-class=\"{'text-red': travelTimes.support.trebuchet.valueType == 'invalid', 'text-green-bright': travelTimes.support.trebuchet.valueType == 'valid'}\" tooltip=\"\" tooltip-content=\"{{ 'travel_time_relocate' | i18n:loc.ale:'military_operations' }}\">{{ travelTimes.relocate.trebuchet.value }}</div>
																																																																																																																																																																																																																																																																																																										</div>
																																																																																																																																																																																																																																																																																																									</table>
																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																								<h5 class=\"twx-section\">{{ 'units' | i18n:loc.ale:'common' }}</h5>
																																																																																																																																																																																																																																																																																																								<table class=\"tbl-border-light tbl-striped\">
																																																																																																																																																																																																																																																																																																									<col width=\"25%\">
																																																																																																																																																																																																																																																																																																										<col width=\"25%\">
																																																																																																																																																																																																																																																																																																											<col width=\"25%\">
																																																																																																																																																																																																																																																																																																												<col width=\"25%\">
																																																																																																																																																																																																																																																																																																													<tbody class=\"add-units\">
																																																																																																																																																																																																																																																																																																														<tr>
																																																																																																																																																																																																																																																																																																															<td colspan=\"4\" class=\"actions\">
																																																																																																																																																																																																																																																																																																																<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																																																																																	<li>
																																																																																																																																																																																																																																																																																																																		<div select=\"\" list=\"presets\" selected=\"selectedInsertPreset\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																		<li>
																																																																																																																																																																																																																																																																																																																			<a class=\"clear-units btn btn-orange\" ng-click=\"cleanUnitInputs()\">{{ 'add_clear' | i18n:loc.ale:'command_queue' }}</a>
																																																																																																																																																																																																																																																																																																																		</ul>
																																																																																																																																																																																																																																																																																																																		<tr ng-repeat=\"i in [] | range:(unitOrder.length / 4);\">
																																																																																																																																																																																																																																																																																																																			<td class=\"cell-space-left\">
																																																																																																																																																																																																																																																																																																																				<span class=\"icon-bg-black\" ng-class=\"'icon-34x34-unit-' + unitOrder[i * 4]\" tooltip=\"\" tooltip-content=\"{{ unitOrder[i * 4] | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																																				<input remove-zero=\"\" ng-model=\"commandData.units[unitOrder[i * 4]]\" maxlength=\"5\" placeholder=\"{{ commandData.units[unitOrder[i * 4]] }}\" ng-focus=\"onUnitInputFocus(unitOrder[i * 4])\" ng-blur=\"onUnitInputBlur(unitOrder[i * 4])\">
																																																																																																																																																																																																																																																																																																																					<td class=\"cell-space-left\">
																																																																																																																																																																																																																																																																																																																						<span class=\"icon-bg-black\" ng-class=\"'icon-34x34-unit-' + unitOrder[i * 4 + 1]\" tooltip=\"\" tooltip-content=\"{{ unitOrder[i * 4 + 1] | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																																						<input remove-zero=\"\" ng-model=\"commandData.units[unitOrder[i * 4 + 1]]\" maxlength=\"5\" placeholder=\"{{ commandData.units[unitOrder[i * 4 + 1]] }}\" ng-focus=\"onUnitInputFocus(unitOrder[i * 4 + 1])\" ng-blur=\"onUnitInputBlur(unitOrder[i * 4 + 1])\">
																																																																																																																																																																																																																																																																																																																							<td class=\"cell-space-left\">
																																																																																																																																																																																																																																																																																																																								<span class=\"icon-bg-black\" ng-class=\"'icon-34x34-unit-' + unitOrder[i * 4 + 2]\" tooltip=\"\" tooltip-content=\"{{ unitOrder[i * 4 + 2] | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																																								<input remove-zero=\"\" ng-model=\"commandData.units[unitOrder[i * 4 + 2]]\" maxlength=\"5\" placeholder=\"{{ commandData.units[unitOrder[i * 4 + 2]] }}\" ng-focus=\"onUnitInputFocus(unitOrder[i * 4 + 2])\" ng-blur=\"onUnitInputBlur(unitOrder[i * 4 + 2])\">
																																																																																																																																																																																																																																																																																																																									<td class=\"cell-space-left\">
																																																																																																																																																																																																																																																																																																																										<span class=\"icon-bg-black\" ng-class=\"'icon-34x34-unit-' + unitOrder[i * 4 + 3]\" tooltip=\"\" tooltip-content=\"{{ unitOrder[i * 4 + 3] | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																																										<input remove-zero=\"\" ng-model=\"commandData.units[unitOrder[i * 4 + 3]]\" maxlength=\"5\" placeholder=\"{{ commandData.units[unitOrder[i * 4 + 3]] }}\" ng-focus=\"onUnitInputFocus(unitOrder[i * 4 + 3])\" ng-blur=\"onUnitInputBlur(unitOrder[i * 4 + 3])\">
																																																																																																																																																																																																																																																																																																																											<tr>
																																																																																																																																																																																																																																																																																																																												<td class=\"cell-space-left\">
																																																																																																																																																																																																																																																																																																																													<span class=\"icon-bg-black icon-34x34-unit-catapult\" tooltip=\"\" tooltip-content=\"{{ 'catapult' | i18n:loc.ale:'unit_names' }}\"/>
																																																																																																																																																																																																																																																																																																																													<input remove-zero=\"\" ng-model=\"commandData.units.catapult\" maxlength=\"5\" placeholder=\"{{ commandData.units.catapult }}\" ng-keyup=\"catapultTargetVisibility()\" ng-focus=\"onUnitInputFocus('catapult')\" ng-blur=\"onUnitInputBlur('catapult')\">
																																																																																																																																																																																																																																																																																																																														<td class=\"cell-space-left\" colspan=\"3\">
																																																																																																																																																																																																																																																																																																																															<div ng-visible=\"showCatapultSelect\">
																																																																																																																																																																																																																																																																																																																																<div class=\"unit-border box-slider\">
																																																																																																																																																																																																																																																																																																																																	<div class=\"height-wrapper\">
																																																																																																																																																																																																																																																																																																																																		<div select=\"\" list=\"attackableBuildings\" selected=\"catapultTarget\"/>
																																																																																																																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																																																																																																																</div>
																																																																																																																																																																																																																																																																																																																															</div>
																																																																																																																																																																																																																																																																																																																														</table>
																																																																																																																																																																																																																																																																																																																														<h5 class=\"twx-section\">{{ 'officers' | i18n:loc.ale:'common' }}</h5>
																																																																																																																																																																																																																																																																																																																														<table class=\"add-officers margin-top tbl-border-light tbl-officers\">
																																																																																																																																																																																																																																																																																																																															<tr>
																																																																																																																																																																																																																																																																																																																																<td class=\"cell-officers\" ng-repeat=\"officer in officers\">
																																																																																																																																																																																																																																																																																																																																	<table class=\"tbl-border-dark tbl-officer\">
																																																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																																																			<td class=\"cell-space\">
																																																																																																																																																																																																																																																																																																																																				<span class=\"icon-44x44-premium_officer_{{ officer }}\"/>
																																																																																																																																																																																																																																																																																																																																				<td class=\"cell-officer-switch\" rowspan=\"2\">
																																																																																																																																																																																																																																																																																																																																					<div switch-slider=\"\" enabled=\"true\" value=\"commandData.officers[officer]\" vertical=\"true\" size=\"'34x66'\"/>
																																																																																																																																																																																																																																																																																																																																					<tr>
																																																																																																																																																																																																																																																																																																																																						<td tooltip=\"\" tooltip-content=\"{{ 'available_officers' | i18n:loc.ale:'modal_preset_edit' }}\">
																																																																																																																																																																																																																																																																																																																																							<div class=\"amount\">{{ inventory.getItemAmountByType('premium_officer_' + officer) | number }}</div>
																																																																																																																																																																																																																																																																																																																																						</table>
																																																																																																																																																																																																																																																																																																																																					</table>
																																																																																																																																																																																																																																																																																																																																				</form>
																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																			<div class=\"waiting rich-text\" ng-show=\"selectedTab === TAB_TYPES.WAITING\">
																																																																																																																																																																																																																																																																																																																																				<div class=\"filters\">
																																																																																																																																																																																																																																																																																																																																					<table class=\"tbl-border-light\">
																																																																																																																																																																																																																																																																																																																																						<tr>
																																																																																																																																																																																																																																																																																																																																							<td>
																																																																																																																																																																																																																																																																																																																																								<div ng-class=\"{'active': activeFilters[FILTER_TYPES.SELECTED_VILLAGE]}\" ng-click=\"toggleFilter(FILTER_TYPES.SELECTED_VILLAGE)\" class=\"box-border-dark icon selectedVillage\" tooltip=\"\" tooltip-content=\"{{ 'filters_selected_village' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-34x34-village-info icon-bg-black\"/>
																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																								<div ng-class=\"{'active': activeFilters[FILTER_TYPES.BARBARIAN_TARGET]}\" ng-click=\"toggleFilter(FILTER_TYPES.BARBARIAN_TARGET)\" class=\"box-border-dark icon barbarianTarget\" tooltip=\"\" tooltip-content=\"{{ 'filters_barbarian_target' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-34x34-barbarian-village icon-bg-black\"/>
																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																								<div ng-class=\"{'active': activeFilters[FILTER_TYPES.ATTACK]}\" ng-click=\"toggleFilter(FILTER_TYPES.ATTACK, true)\" class=\"box-border-dark icon allowedTypes\" tooltip=\"\" tooltip-content=\"{{ 'filters_attack' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-34x34-attack icon-bg-black\"/>
																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																								<div ng-class=\"{'active': activeFilters[FILTER_TYPES.SUPPORT]}\" ng-click=\"toggleFilter(FILTER_TYPES.SUPPORT, true)\" class=\"box-border-dark icon allowedTypes\" tooltip=\"\" tooltip-content=\"{{ 'filters_support' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-34x34-support icon-bg-black\"/>
																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																								<div ng-class=\"{'active': activeFilters[FILTER_TYPES.RELOCATE]}\" ng-click=\"toggleFilter(FILTER_TYPES.RELOCATE, true)\" class=\"box-border-dark icon allowedTypes\" tooltip=\"\" tooltip-content=\"{{ 'filters_relocate' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-34x34-relocate icon-bg-black\"/>
																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																								<div class=\"text\">
																																																																																																																																																																																																																																																																																																																																									<input ng-model=\"filtersData[FILTER_TYPES.TEXT_MATCH]\" class=\"box-border-dark\" placeholder=\"{{ 'filters_text_match' | i18n:loc.ale:'command_queue' }}\">
																																																																																																																																																																																																																																																																																																																																									</div>
																																																																																																																																																																																																																																																																																																																																								</table>
																																																																																																																																																																																																																																																																																																																																							</div>
																																																																																																																																																																																																																																																																																																																																							<div class=\"queue\">
																																																																																																																																																																																																																																																																																																																																								<h5 class=\"twx-section\">{{ 'queue_waiting' | i18n:loc.ale:'command_queue' }}</h5>
																																																																																																																																																																																																																																																																																																																																								<p class=\"text-center\" ng-show=\"!visibleWaitingCommands.length\">{{ 'queue_none_added' | i18n:loc.ale:'command_queue' }}<table class=\"tbl-border-light\" ng-repeat=\"command in visibleWaitingCommands\">
																																																																																																																																																																																																																																																																																																																																										<col width=\"100px\">
																																																																																																																																																																																																																																																																																																																																											<tr>
																																																																																																																																																																																																																																																																																																																																												<th colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																													<span ng-class=\"{true: 'icon-bg-red', false:'icon-bg-blue'}[command.type === COMMAND_TYPES.ATTACK]\" class=\"icon-26x26-{{ command.type }}\" tooltip=\"\" tooltip-content=\"{{ command.type | i18n:loc.ale:'common' }}\"/>
																																																																																																																																																																																																																																																																																																																																													<span class=\"size-26x26 icon-bg-black icon-26x26-time-duration\" tooltip=\"\" tooltip-content=\"{{ 'command_time_left' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																													<span class=\"time-left\">{{ command.countdown() }}</span>
																																																																																																																																																																																																																																																																																																																																													<span class=\"size-26x26 icon-bg-black icon-20x20-units-outgoing\" tooltip=\"\" tooltip-content=\"{{ 'command_out' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																													<span class=\"sent-time\">{{ command.sendTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																													<span class=\"size-26x26 icon-bg-black icon-20x20-time-arrival\" tooltip=\"\" tooltip-content=\"{{ 'command_arrive' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																													<span class=\"arrive-time\">{{ command.arriveTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																													<a href=\"#\" class=\"remove-command size-20x20 btn-red icon-20x20-close\" ng-click=\"removeCommand(command, EVENT_CODES.COMMAND_REMOVED)\" tooltip=\"\" tooltip-content=\"{{ 'queue_remove' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																													<tr>
																																																																																																																																																																																																																																																																																																																																														<td>{{ 'villages' | i18n:loc.ale:'common' }}<td>
																																																																																																																																																																																																																																																																																																																																																<a class=\"origin\">
																																																																																																																																																																																																																																																																																																																																																	<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.origin.id)\">{{ command.origin.name }} ({{ command.origin.x }}|{{ command.origin.y }})</span>
																																																																																																																																																																																																																																																																																																																																																</a>
																																																																																																																																																																																																																																																																																																																																																<span class=\"size-20x20 icon-26x26-{{ command.type }}\"/>
																																																																																																																																																																																																																																																																																																																																																<a class=\"target\">
																																																																																																																																																																																																																																																																																																																																																	<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.target.id)\">{{ command.target.name }} ({{ command.target.x }}|{{ command.target.y }})</span>
																																																																																																																																																																																																																																																																																																																																																</a>
																																																																																																																																																																																																																																																																																																																																																<tr>
																																																																																																																																																																																																																																																																																																																																																	<td>{{ 'units' | i18n:loc.ale:'common' }}<td class=\"units\">
																																																																																																																																																																																																																																																																																																																																																			<div class=\"unit\" ng-repeat=\"(unit, amount) in command.units\">
																																																																																																																																																																																																																																																																																																																																																				<span class=\"icon-34x34-unit-{{ unit }} icon\"/>
																																																																																																																																																																																																																																																																																																																																																				<span class=\"amount\">{{ amount }}</span>
																																																																																																																																																																																																																																																																																																																																																				<span ng-if=\"unit === 'catapult' && command.type === COMMAND_TYPES.ATTACK\">({{ command.catapultTarget | i18n:loc.ale:'common' }})</span>
																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																			<div class=\"officer\" ng-repeat=\"(officer, enabled) in command.officers\">
																																																																																																																																																																																																																																																																																																																																																				<span class=\"icon-34x34-premium_officer_{{ officer }}\"/>
																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																		</table>
																																																																																																																																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																																																																																																																																</div>
																																																																																																																																																																																																																																																																																																																																																<div class=\"logs rich-text\" ng-show=\"selectedTab === TAB_TYPES.LOGS\">
																																																																																																																																																																																																																																																																																																																																																	<h5 class=\"twx-section\">{{ 'queue_sent' | i18n:loc.ale:'command_queue' }}</h5>
																																																																																																																																																																																																																																																																																																																																																	<p class=\"text-center\" ng-show=\"!sentCommands.length\">{{ 'queue_none_sent' | i18n:loc.ale:'command_queue' }}<table class=\"tbl-border-light\" ng-repeat=\"command in sentCommands track by $index\">
																																																																																																																																																																																																																																																																																																																																																			<col width=\"100px\">
																																																																																																																																																																																																																																																																																																																																																				<tr>
																																																																																																																																																																																																																																																																																																																																																					<th colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																						<span ng-class=\"{true: 'icon-bg-red', false:'icon-bg-blue'}[command.type === COMMAND_TYPES.ATTACK]\" class=\"icon-26x26-{{ command.type }}\" tooltip=\"\" tooltip-content=\"{{ command.type | i18n:loc.ale:'common' }}\"/>
																																																																																																																																																																																																																																																																																																																																																						<span class=\"size-26x26 icon-bg-black icon-20x20-units-outgoing\" tooltip=\"\" tooltip-content=\"{{ 'command_out' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																																						<span class=\"sent-time\">{{ command.sendTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																																						<span class=\"size-26x26 icon-bg-black icon-20x20-time-arrival\" tooltip=\"\" tooltip-content=\"{{ 'command_arrive' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																																						<span class=\"arrive-time\">{{ command.arriveTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																																						<tr>
																																																																																																																																																																																																																																																																																																																																																							<td>{{ 'villages' | i18n:loc.ale:'common' }}<td>
																																																																																																																																																																																																																																																																																																																																																									<a class=\"origin\">
																																																																																																																																																																																																																																																																																																																																																										<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.origin.id)\">{{ command.origin.name }} ({{ command.origin.x }}|{{ command.origin.y }})</span>
																																																																																																																																																																																																																																																																																																																																																									</a>
																																																																																																																																																																																																																																																																																																																																																									<span class=\"size-20x20 icon-26x26-{{ command.type }}\"/>
																																																																																																																																																																																																																																																																																																																																																									<a class=\"target\">
																																																																																																																																																																																																																																																																																																																																																										<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.target.id)\">{{ command.target.name }} ({{ command.target.x }}|{{ command.target.y }})</span>
																																																																																																																																																																																																																																																																																																																																																									</a>
																																																																																																																																																																																																																																																																																																																																																									<tr>
																																																																																																																																																																																																																																																																																																																																																										<td>{{ 'units' | i18n:loc.ale:'common' }}<td class=\"units\">
																																																																																																																																																																																																																																																																																																																																																												<div class=\"unit\" ng-repeat=\"(unit, amount) in command.units\">
																																																																																																																																																																																																																																																																																																																																																													<span class=\"icon-34x34-unit-{{ unit }} icon\"/>
																																																																																																																																																																																																																																																																																																																																																													<span class=\"amount\">{{ amount }}</span>
																																																																																																																																																																																																																																																																																																																																																													<span ng-if=\"unit === 'catapult' && command.type === COMMAND_TYPES.ATTACK\">({{ command.catapultTarget | i18n:loc.ale:'common' }})</span>
																																																																																																																																																																																																																																																																																																																																																												</div>
																																																																																																																																																																																																																																																																																																																																																												<div class=\"officer\" ng-repeat=\"(officer, enabled) in command.officers\">
																																																																																																																																																																																																																																																																																																																																																													<span class=\"icon-34x34-premium_officer_{{ officer }}\"/>
																																																																																																																																																																																																																																																																																																																																																												</div>
																																																																																																																																																																																																																																																																																																																																																											</table>
																																																																																																																																																																																																																																																																																																																																																											<h5 class=\"twx-section\">{{ 'queue_expired' | i18n:loc.ale:'command_queue' }}</h5>
																																																																																																																																																																																																																																																																																																																																																											<p class=\"text-center\" ng-show=\"!expiredCommands.length\">{{ 'queue_none_expired' | i18n:loc.ale:'command_queue' }}<table class=\"tbl-border-light\" ng-repeat=\"command in expiredCommands track by $index\">
																																																																																																																																																																																																																																																																																																																																																													<col width=\"100px\">
																																																																																																																																																																																																																																																																																																																																																														<tr>
																																																																																																																																																																																																																																																																																																																																																															<th colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																																<span ng-class=\"{true: 'icon-bg-red', false:'icon-bg-blue'}[command.type === COMMAND_TYPES.ATTACK]\" class=\"icon-26x26-{{ command.type }}\" tooltip=\"\" tooltip-content=\"{{ command.type | i18n:loc.ale:'common' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																<span class=\"size-26x26 icon-bg-black icon-20x20-units-outgoing\" tooltip=\"\" tooltip-content=\"{{ 'command_out' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																<span class=\"sent-time\">{{ command.sendTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																																																<span class=\"size-26x26 icon-bg-black icon-20x20-time-arrival\" tooltip=\"\" tooltip-content=\"{{ 'command_arrive' | i18n:loc.ale:'command_queue' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																<span class=\"arrive-time\">{{ command.arriveTime | readableDateFilter:loc.ale }}</span>
																																																																																																																																																																																																																																																																																																																																																																<tr>
																																																																																																																																																																																																																																																																																																																																																																	<td>{{ 'villages' | i18n:loc.ale:'common' }}<td>
																																																																																																																																																																																																																																																																																																																																																																			<a class=\"origin\">
																																																																																																																																																																																																																																																																																																																																																																				<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.origin.id)\">{{ command.origin.name }} ({{ command.origin.x }}|{{ command.origin.y }})</span>
																																																																																																																																																																																																																																																																																																																																																																			</a>
																																																																																																																																																																																																																																																																																																																																																																			<span class=\"size-20x20 icon-26x26-{{ command.type }}\"/>
																																																																																																																																																																																																																																																																																																																																																																			<a class=\"target\">
																																																																																																																																																																																																																																																																																																																																																																				<span class=\"village-link img-link icon-20x20-village btn btn-orange padded\" ng-click=\"openVillageInfo(command.target.id)\">{{ command.target.name }} ({{ command.target.x }}|{{ command.target.y }})</span>
																																																																																																																																																																																																																																																																																																																																																																			</a>
																																																																																																																																																																																																																																																																																																																																																																			<tr>
																																																																																																																																																																																																																																																																																																																																																																				<td>{{ 'units' | i18n:loc.ale:'common' }}<td class=\"units\">
																																																																																																																																																																																																																																																																																																																																																																						<div class=\"unit\" ng-repeat=\"(unit, amount) in command.units\">
																																																																																																																																																																																																																																																																																																																																																																							<span class=\"icon-34x34-unit-{{ unit }} icon\"/>
																																																																																																																																																																																																																																																																																																																																																																							<span class=\"amount\">{{ amount }}</span>
																																																																																																																																																																																																																																																																																																																																																																							<span ng-if=\"unit === 'catapult' && command.type === COMMAND_TYPES.ATTACK\">({{ command.catapultTarget | i18n:loc.ale:'common' }})</span>
																																																																																																																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																																																																																																																						<div class=\"officer\" ng-repeat=\"(officer, enabled) in command.officers\">
																																																																																																																																																																																																																																																																																																																																																																							<span class=\"icon-34x34-premium_officer_{{ officer }}\"/>
																																																																																																																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																																																																																																																					</table>
																																																																																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																																																																																																																																																	<footer class=\"win-foot\">
																																																																																																																																																																																																																																																																																																																																																																		<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																																																																																																																																			<li ng-show=\"selectedTab === TAB_TYPES.LOGS\">
																																																																																																																																																																																																																																																																																																																																																																				<a class=\"btn-orange btn-border\" ng-click=\"clearRegisters()\">{{ 'general_clear' | i18n:loc.ale:'command_queue' }}</a>
																																																																																																																																																																																																																																																																																																																																																																				<li ng-show=\"selectedTab === TAB_TYPES.ADD\">
																																																																																																																																																																																																																																																																																																																																																																					<a class=\"btn-orange btn-border add\" ng-click=\"addCommand(COMMAND_TYPES.ATTACK)\">
																																																																																																																																																																																																																																																																																																																																																																						<span class=\"icon-26x26-attack-small\"/> {{ COMMAND_TYPES.ATTACK | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																					<li ng-show=\"selectedTab === TAB_TYPES.ADD\">
																																																																																																																																																																																																																																																																																																																																																																						<a class=\"btn-orange btn-border add\" ng-click=\"addCommand(COMMAND_TYPES.SUPPORT)\">
																																																																																																																																																																																																																																																																																																																																																																							<span class=\"icon-26x26-support\"/> {{ COMMAND_TYPES.SUPPORT | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																						<li ng-show=\"selectedTab === TAB_TYPES.ADD\">
																																																																																																																																																																																																																																																																																																																																																																							<a class=\"btn-orange btn-border add\" ng-click=\"addCommand(COMMAND_TYPES.RELOCATE)\">
																																																																																																																																																																																																																																																																																																																																																																								<span class=\"icon-26x26-relocate\"/> {{ COMMAND_TYPES.RELOCATE | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																							<li>
																																																																																																																																																																																																																																																																																																																																																																								<a href=\"#\" ng-class=\"{false:'btn-green', true:'btn-red'}[running]\" class=\"btn-border\" ng-click=\"switchCommandQueue()\">
																																																																																																																																																																																																																																																																																																																																																																									<span ng-show=\"running\">{{ 'deactivate' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																																																																																																																																																									<span ng-show=\"!running\">{{ 'activate' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																																																																																																																																																								</a>
																																																																																																																																																																																																																																																																																																																																																																							</ul>
																																																																																																																																																																																																																																																																																																																																																																						</footer>
																																																																																																																																																																																																																																																																																																																																																																					</div>`)
        interfaceOverflow.addStyle('#two-command-queue input.unit{width:80px;height:34px}#two-command-queue form .padded{padding:2px 8px}#two-command-queue .basic-config input{width:200px;font-weight:bold;padding:5px 8px;outline:none;border:none;color:#000;resize:none}#two-command-queue a.select-handler{-webkit-box-shadow:none;box-shadow:none;height:22px;line-height:22px}#two-command-queue a.select-button{height:22px}#two-command-queue .custom-select{width:240px}#two-command-queue .originVillage,#two-command-queue .targetVillage{padding:0 7px}#two-command-queue a.btn{height:28px;line-height:28px;padding:0 10px}#two-command-queue .actions{text-align:center;user-select:none}#two-command-queue .add-units td{text-align:center}#two-command-queue .add-units .unit-icon{top:-1px}#two-command-queue .add-units input{height:34px;line-height:26px;color:#fff3d0;border:none;outline:none;font-size:14px;background:url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAMAAAAp4XiDAAAABGdBTUEAALGPC/xhBQAAALRQTFRFr6+vmJiYoKCgrKysq6urpaWltLS0s7OzsLCwpKSkm5ubqKiojY2NlZWVk5OTqampbGxsWFhYUVFRhISEgYGBmpqaUFBQnp6eYmJidnZ2nZ2dY2NjW1tbZ2dnoaGhe3t7l5eXg4ODVVVVWVlZj4+PXFxcVlZWkpKSZmZmdXV1ZWVlc3NzjIyMXl5eVFRUeHh4hoaGYWFhXV1dbW1tampqb29veXl5fHx8gICAiYmJcnJyTk5Ooj6l1wAAADx0Uk5TGhkZGhoaGxoaGRkaGRkZGhkbHBgYGR0ZGhkZGhsZGRgZGRwbGRscGRoZGhkZGhwZGRobGRkZGRkZGRkeyXExWQAABOJJREFUSMeNVgdy4zgQxIW9TQ7KOVEUo5gz0f//1/WA0sple6+OLokQiUk9PQ2rvlzvT0vA6xDXU3R5hQmqddDVaIELsMl3KLUGoFHugUphjt25PWkE6KMAqPkO/Qh7HRadPmTNxKJpWuhSjLZAoSZmXYoPXh0w2R2z10rjBxpMNRfomhbNFUfUFbfUCh6TWmO4ZqNn6Jxekx6lte3h9IgYv9ZwzIZXfhQ/bejmsYkgOeVInoDGT6KGP9MMbsj7mtEKphKgVFKkJGUM+r/00zybNkPMFWYske+jY9hUblbrK4YosyPtrxl+5kNRWSb2B3+pceKT05SQRPZY8pVSGoWutgen2junRVKPZJ0v5Nu9HAk/CFPr+T1XTkXYFWSJXfTyLPcpcPXtBZIPONq/cFQ0Y0Lr1GF6f5doHdm2RLTbQMpMmCIf/HGm53OLFPiiEOsBKtgHccgKTVwn8l7kbt3iPvqniMX4jgWj4aqlX43xLwXVet5XTG1cYp/29m58q6ULSa7V0M3UQFyjd+AD+1W9WLBpDd9uej7emFbea/+Yw8faySElQQrBDksTpTOVIG/SE2HpPvZsplJWsblRLEGXATEW9YLUY1rPSdivBDmuK3exNiAysfPALfYZFWJrsA4Zt+fftEeRY0UsMDqfyNCKJpdrtI1r2k0vp9LMSwdO0u5SpjBeEYz5ebhWNbwT2g7OJXy1vjW+pEwyd1FTkAtbzzcbmX1yZlkR2pPiXZ/mDbPNWvHRsaKfLH8+FqiZbnodbOK9RGWlNMli8k+wsgbSNwS35QB6qxn53xhu2DFqUilisB9q2Zqw4nNI9tOB2z8GbkvEdNjPaD2j+9pwEC+YlWJvI7xN7xMC09eqhq/qwRvz3JWcFWmkjrWBWSiOysEmc4LmMb0iSsxR8+Z8pk3+oE39cdAmh1xSDXuAryRLZgpp9V62+8IOeBSICjs8LlbtKGN4E7XGoGASIJ+vronVa5mjagPHIFJA2b+BKkZC5I/78wOqmzYp1N8vzTkWIWz6YfsS3eh3w8pBkfKz6TSLxK9Qai5DUGTMZ8NNmrW8ldNudIJq+eJycwjv+xbeOJwPv1jjsSV/rCBaS/IBrafaUQ+5ksHwwl9y9X7kmvvIKWoBDFvbWySGyMU3XflxZRkNeRU63otWb0+P8H8BrRokbJivpWkk6m6LccSlrC2K0i6+4otx4dN3mbAVKt0wbaqBab4/MW8rgrS8JP06HU6UYSTYsQ5pYETpo87ZonORvbPlvYbXwmsMgoQGKr8PUQ5dDEO0EcXp2oOfSk+YpR/Eg4R46O0/Sf7jVnbqbXBrRkCPsZFOQTN8h+aqlcRw9FjJ/j8V7SXZ3hVNXYsOYcxzpfPNgFrvB9S6Dej2PqDqq0su+5ng0WMi527p/pA+OiW0fsYzDa6sPS9C1qxTtxVRMuySrwPD6qGPRKc4uIx4oceJ9FPjxWaqPPebzyXxU7W1jNqqOw+9z6X/k+Na3SBa0v+VjgoaULR30G1nxvZN1vsha2UaSrKy/PyCaHK5zAYnJzm9RSpSPDWbDVu0dkUujMmB/ly4w8EnDdXXoyX/VfhB3yKzMJ2BSaZO+A9GiNQMbll+6z1WGLWpEGMeEg85MESSep0IPFaHYZZ1QOW/xcjfxGhNjP0tRtbhFHOmhhjAv/p77JrCX3+ZAAAAAElFTkSuQmCC) top left #b89064;box-shadow:inset 0 0 0 1px #000,inset 0 0 0 2px #a2682c,inset 0 0 0 3px #000,inset -3px -3px 2px 0 #fff,inset 0 0 9px 5px rgba(99,54,0,0.5);text-align:center;width:80px}#two-command-queue .add-officers .cell-officers{padding:7px 11px 5px 11px}#two-command-queue .add-officers .amount{color:#fff;text-align:center}#two-command-queue .command{margin-bottom:10px}#two-command-queue .command .time-left{width:93px;display:inline-block;padding:0 0 0 3px}#two-command-queue .command .sent-time,#two-command-queue .command .arrive-time{width:160px;display:inline-block;padding:0 0 0 5px}#two-command-queue .command td{padding:3px 6px}#two-command-queue .officers td{width:111px;text-align:center}#two-command-queue .officers label{margin-left:5px}#two-command-queue .officers span{margin-left:2px}#two-command-queue .units div.unit{float:left}#two-command-queue .units div.unit span.icon{transform:scale(.7);width:25px;height:25px}#two-command-queue .units div.unit span.amount{vertical-align:-2px;margin:0 5px 0 2px}#two-command-queue .units div.officer{float:left;margin:0 2px}#two-command-queue .units div.officer span{transform:scale(.7);width:25px;height:25px}#two-command-queue .remove-command{float:right;margin-top:3px}#two-command-queue .tbl-units td{text-align:center}#two-command-queue .tbl-speed{margin-top:10px}#two-command-queue .tbl-speed th{text-align:center}#two-command-queue .tbl-speed td{font-size:12px}#two-command-queue .tbl-speed .box-time-sub-icon{position:relative}#two-command-queue .tbl-speed .box-time-sub-icon .time-icon{position:absolute;top:-3px;left:27px;transform:scale(.7)}#two-command-queue .tbl-speed .box-time-sub-icon.time-relocate .time-icon{top:-6px;left:26px}#two-command-queue .dateType{width:200px}#two-command-queue .dateType .custom-select-handler{text-align:left}#two-command-queue .filters .icon{width:38px;float:left;margin:0 6px}#two-command-queue .filters .icon.active:before{box-shadow:0 0 0 1px #000,-1px -1px 0 2px #ac9c44,0 0 0 3px #ac9c44,0 0 0 4px #000;border-radius:1px;content:"";position:absolute;width:38px;height:38px;left:-1px;top:-1px}#two-command-queue .filters .text{margin-left:262px}#two-command-queue .filters .text input{height:36px;margin-top:1px;width:100%;text-align:left;padding:0 5px}#two-command-queue .filters .text input::placeholder{color:white}#two-command-queue .filters .text input:focus::placeholder{color:transparent}#two-command-queue .filters td{padding:6px}#two-command-queue .icon-34x34-barbarian-village:before{filter:grayscale(100%);background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-220px -906px}#two-command-queue .icon-20x20-time-arrival:before{transform:scale(.8);background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-529px -454px}#two-command-queue .icon-20x20-attack:before{transform:scale(.8);background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-546px -1086px;width:26px;height:26px}#two-command-queue .icon-20x20-support:before{transform:scale(.8);background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-462px -360px;width:26px;height:26px}#two-command-queue .icon-20x20-relocate:before{transform:scale(.8);background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-1090px -130px;width:26px;height:26px}#two-command-queue .icon-26x26-attack:before{background-image:url(https://i.imgur.com/ozI4k0h.png);background-position:-546px -1086px}')
    }

    const buildWindow = function () {
        const lastDateType = Lockr.get(STORAGE_KEYS.LAST_DATE_TYPE, DATE_TYPES.OUT, true)

        $scope = $rootScope.$new()
        $scope.selectedTab = DEFAULT_TAB
        $scope.inventory = modelDataService.getInventory()
        $scope.presets = utils.obj2selectOptions(presetList.getPresets())
        $scope.travelTimes = {}
        $scope.unitOrder = unitOrder
        $scope.officers = $gameData.getOrderedOfficerNames()
        $scope.searchQuery = {
            origin: '',
            target: ''
        }
        $scope.isValidDate = false
        $scope.dateTypes = util.toActionList(DATE_TYPES, function (actionType) {
            return $filter('i18n')(actionType, $rootScope.loc.ale, 'command_queue')
        })
        $scope.selectedDateType = {
            name: $filter('i18n')(lastDateType, $rootScope.loc.ale, 'command_queue'),
            value: lastDateType
        }
        $scope.selectedInsertPreset = {
            name: $filter('i18n')('add_insert_preset', $rootScope.loc.ale, 'command_queue'),
            value: null
        }
        $scope.catapultTarget = {
            name: $filter('i18n')(DEFAULT_CATAPULT_TARGET, $rootScope.loc.ale, 'building_names'),
            value: DEFAULT_CATAPULT_TARGET
        }
        $scope.autoCompleteOrigin = {
            type: ['village'],
            placeholder: $filter('i18n')('add_village_search', $rootScope.loc.ale, 'command_queue'),
            onEnter: eventHandlers.onAutoCompleteOrigin,
            tooltip: $filter('i18n')('add_origin', $rootScope.loc.ale, 'command_queue'),
            dropDown: true
        }
        $scope.autoCompleteTarget = {
            type: ['village'],
            placeholder: $filter('i18n')('add_village_search', $rootScope.loc.ale, 'command_queue'),
            onEnter: eventHandlers.onAutoCompleteTarget,
            tooltip: $filter('i18n')('add_target', $rootScope.loc.ale, 'command_queue'),
            dropDown: true
        }
        $scope.showCatapultSelect = !!commandData.units.catapult
        $scope.attackableBuildings = attackableBuildingsList
        $scope.commandData = commandData
        $scope.activeFilters = activeFilters
        $scope.filtersData = filtersData
        $scope.running = commandQueue.isRunning()
        $scope.waitingCommands = commandQueue.getWaitingCommands()
        $scope.visibleWaitingCommands = commandQueue.getWaitingCommands()
        $scope.sentCommands = commandQueue.getSentCommands()
        $scope.expiredCommands = commandQueue.getExpiredCommands()
        $scope.EVENT_CODES = EVENT_CODES
        $scope.FILTER_TYPES = FILTER_TYPES
        $scope.TAB_TYPES = TAB_TYPES
        $scope.COMMAND_TYPES = COMMAND_TYPES

        // functions
        $scope.onUnitInputFocus = onUnitInputFocus
        $scope.onUnitInputBlur = onUnitInputBlur
        $scope.catapultTargetVisibility = catapultTargetVisibility
        $scope.selectTab = selectTab
        $scope.addSelected = addSelected
        $scope.addMapSelected = addMapSelected
        $scope.addCurrentDate = addCurrentDate
        $scope.incrementDate = incrementDate
        $scope.reduceDate = reduceDate
        $scope.cleanUnitInputs = cleanUnitInputs
        $scope.addCommand = addCommand
        $scope.clearRegisters = clearRegisters
        $scope.switchCommandQueue = switchCommandQueue
        $scope.removeCommand = commandQueue.removeCommand
        $scope.openVillageInfo = windowDisplayService.openVillageInfo
        $scope.toggleFilter = toggleFilter

        $scope.$watch('commandData.origin', updateTravelTimes)
        $scope.$watch('commandData.target', updateTravelTimes)
        $scope.$watch('commandData.date', updateTravelTimes)
        $scope.$watch('selectedDateType.value', updateDateType)
        $scope.$watch('selectedInsertPreset.value', insertPreset)
        $scope.$watch('filtersData[FILTER_TYPES.TEXT_MATCH]', textMatchFilter)

        let eventScope = new EventScope('twoverflow_queue_window')
        eventScope.register(eventTypeProvider.ARMY_PRESET_UPDATE, eventHandlers.updatePresets, true)
        eventScope.register(eventTypeProvider.ARMY_PRESET_DELETED, eventHandlers.updatePresets, true)
        eventScope.register(eventTypeProvider.SELECT_SELECTED, eventHandlers.autoCompleteSelected, true)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_ORIGIN, eventHandlers.addInvalidOrigin)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_TARGET, eventHandlers.addInvalidTarget)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD_INVALID_DATE, eventHandlers.addInvalidDate)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD_NO_UNITS, eventHandlers.addNoUnits)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD_ALREADY_SENT, eventHandlers.addAlreadySent)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_REMOVE, eventHandlers.removeCommand)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_REMOVE_ERROR, eventHandlers.removeError)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_SEND_TIME_LIMIT, eventHandlers.sendTimeLimit)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_SEND_NOT_OWN_VILLAGE, eventHandlers.sendNotOwnVillage)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_SEND_NO_UNITS_ENOUGH, eventHandlers.sendNoUnitsEnough)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_ADD, eventHandlers.addCommand)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_SEND, eventHandlers.sendCommand)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_START, eventHandlers.start)
        eventScope.register(eventTypeProvider.COMMAND_QUEUE_STOP, eventHandlers.stop)

        windowManagerService.getScreenWithInjectedScope('!twoverflow_queue_window', $scope)
    }

    return init
})

define('two/commandQueue/storageKeys', [], function () {
    return {
        QUEUE_COMMANDS: 'command_queue_commands',
        QUEUE_SENT: 'command_queue_sent',
        QUEUE_EXPIRED: 'command_queue_expired',
        LAST_DATE_TYPE: 'command_queue_last_date_type'
    }
})

define('two/commandQueue/types/commands', [], function () {
    return {
        'ATTACK': 'attack',
        'SUPPORT': 'support',
        'RELOCATE': 'relocate'
    }
})

define('two/commandQueue/types/dates', [], function () {
    return {
        ARRIVE: 'date_type_arrive',
        OUT: 'date_type_out'
    }
})

define('two/commandQueue/types/events', [], function () {
    return {
        NOT_OWN_VILLAGE: 'not_own_village',
        NOT_ENOUGH_UNITS: 'not_enough_units',
        TIME_LIMIT: 'time_limit',
        COMMAND_REMOVED: 'command_removed',
        COMMAND_SENT: 'command_sent'
    }
})

define('two/commandQueue/types/filters', [], function () {
    return {
        SELECTED_VILLAGE: 'selected_village',
        BARBARIAN_TARGET: 'barbarian_target',
        ALLOWED_TYPES: 'allowed_types',
        ATTACK: 'attack',
        SUPPORT: 'support',
        RELOCATE: 'relocate',
        TEXT_MATCH: 'text_match'
    }
})

require([
    'two/language',
    'two/ready',
    'two/commandQueue',
    'two/commandQueue/ui',
    'two/commandQueue/events'
], function (
    twoLanguage,
    ready,
    commandQueue,
    commandQueueInterface
) {
    if (commandQueue.initialized) {
        return false
    }

    ready(function () {
        twoLanguage.add('command_queue', {"de_de":{"command_queue":{"title":"Befehlsschleife","attack":"angreifen","support":"unterst체tzen","relocate":"umsiedeln","sent":"gesendet","activated":"aktiviert","deactivated":"deaktiviert","expired":"abgelaufen","removed":"entfernt","added":"hinzugef체gt","general_clear":"Protokoll l철schen","general_next_command":"N채chster Befehl","add_basics":"Informationen","add_origin":"Startkoordinaten","add_selected":"Aktives Dorf","add_target":"Zielkoordinaten","add_map_selected":"Ausgew채hltes Dorf","date_type_arrive":"Eintreffzeitpunkt","date_type_out":"Startzeitpunkt","add_current_date":"Jetzt","add_current_date_plus":"Zeitpunkt um 100 Millisekunden erh철hen.","add_current_date_minus":"Zeitpunkt um 100 Millisekunden verringern.","add_travel_times":"Laufzeit","add_date":"Zeitpunkt","add_no_village":"Dorf ausw채hlen...","add_village_search":"Dorf suchen...","add_clear":"Felder leeren","add_insert_preset":"Vorlagen einf체gen","queue_waiting":"Ausstehende Befehle","queue_none_added":"Es wurden keine Befehle hinzugef체gt.","queue_sent":"Befehle gesendet","queue_none_sent":"Kein Befehl gesendet.","queue_expired":"Abgelaufene Befehle","queue_none_expired":"Keine abgelaufenen Befehle.","queue_remove":"Befehl aus Liste entfernen","queue_filters":"Filter","filters_selected_village":"Zeige nur Befehle bez체glich des aktuellen Dorfs","filters_barbarian_target":"Zeige nur Befehle mit Barbarend철rfer als Ziel","filters_attack":"Zeige Angriffe","filters_support":"Zeige Unterst체tzungen","filters_relocate":"Zeige Umsiedlungen","filters_text_match":"Filtern nach Text...","command_out":"Ausgehend","command_time_left":"Zeit verbleibend","command_arrive":"Ankunftszeit","error_no_units_enough":"Nicht genug Einheiten f체r diesen Befehl!","error_not_own_village":"Du bist nicht im Besitz des Startdorfes!","error_origin":"Startkoordinaten ung체ltig!","error_target":"Zielkoordinaten ung체ltig!","error_no_units":"Keine Einheiten ausgew채hlt!","error_invalid_date":"Ung체ltiges Datum","error_already_sent":"Diese %{type} h채tten um %{date} rausgehen m체ssen","error_no_map_selected_village":"Kein Dorf auf der Karte ausgew채hlt.","error_remove_error":"Fehler beim Entfernen des Befehls.","tab_add":"Befehl hinzuf체gen","tab_waiting":"Befehle in Warteschleife","tab_logs":"Befehlsprotokoll"}},"en_us":{"command_queue":{"title":"CommandQueue","attack":"attack","support":"support","relocate":"transfer","sent":"sent","activated":"activated","deactivated":"disabled","expired":"expired","removed":"removed","added":"added","general_clear":"Clear logs","general_next_command":"Next command","add_basics":"Basic information","add_origin":"Origin coordinates","add_selected":"Active village","add_target":"Target coordinates","add_map_selected":"Selected village on a map","date_type_arrive":"Command arrive at date","date_type_out":"Command leave at date","add_current_date":"Current date","add_current_date_plus":"Increase date in 100 milliseconds.","add_current_date_minus":"Reduce date in 100 milliseconds.","add_travel_times":"Travel time","add_date":"Date/time","add_no_village":"select a village...","add_village_search":"Village search...","add_clear":"Clear fields","add_insert_preset":"Insert preset","queue_waiting":"Waiting commands","queue_none_added":"No command added.","queue_sent":"Commands sent","queue_none_sent":"No command sent.","queue_expired":"Expired commands","queue_none_expired":"No command expired.","queue_remove":"Remove command form list","queue_filters":"Commands filter","filters_selected_village":"Show only commands from the selected village","filters_barbarian_target":"Show only commands with barbarian villages as target","filters_attack":"Show attacks","filters_support":"Show supports","filters_relocate":"Show transfers","filters_text_match":"Filter by text...","command_out":"Out","command_time_left":"Time remaining","command_arrive":"Arrival","error_no_units_enough":"No units enough to send the command!","error_not_own_village":"The origin village is not owned by you!","error_origin":"Origin village coordinates invalid!","error_target":"Target village coordinates invalid!","error_no_units":"No units specified!","error_invalid_date":"Invalid date","error_already_sent":"This %d should have left %d","error_no_map_selected_village":"No selected village on the map.","error_remove_error":"Error removing command.","tab_add":"Add command","tab_waiting":"Queued commands","tab_logs":"Command logs"}},"pl_pl":{"command_queue":{"title":"CommandQueue","attack":"Atak","support":"Wsparcie","relocate":"przenie힄","sent":"wys흢any/e","activated":"w흢훳czony","deactivated":"wy흢훳czony","expired":"przedawniony/e","removed":"usuni휌ty/e","added":"dodany/e","general_clear":"Wyczy힄훶 logi","general_next_command":"Nast휌pny rozkaz","add_basics":"Podstawowe informacje","add_origin":"탁r처d흢o","add_selected":"Aktywna wioska","add_target":"Cel","add_map_selected":"Wybrana wioska na mapie","date_type_arrive":"Czas dotarcia na cel","date_type_out":"Czas wyj힄cia z twojej wioski","add_current_date":"Obecny czas","add_current_date_plus":"Zwi휌ksz czas o 100 milisekund.","add_current_date_minus":"Zmniejsz czas o 100 milisekund.","add_travel_times":"Czas podr처탉y jednostek","add_date":"Czas/Data","add_no_village":"Wybierz wiosk휌...","add_village_search":"Znajd탄 wiosk휌...","add_clear":"wyczy힄훶","add_insert_preset":"Wybierz szablon","queue_waiting":"Rozkazy","queue_none_added":"Brak dodanych rozkaz처w.","queue_sent":"Rozkazy wys흢ane","queue_none_sent":"Brak wys흢anych rozkaz처w.","queue_expired":"Przedawnione rozkazy","queue_none_expired":"Brak przedawnionych rozkaz처w.","queue_remove":"Usu흦 rozkaz z listy","queue_filters":"Filtruj rozkazy","filters_selected_village":"Poka탉 tylko rozkazy z aktywnej wioski","filters_barbarian_target":"Poka탉 tylko rozkazy na wioski barbarzy흦skie","filters_attack":"Poka탉 ataki","filters_support":"Poka탉 wsparcia","filters_relocate":"Poka탉 przeniesienia","filters_text_match":"Filtruj za pomoc훳 tekstu...","command_out":"Czas wyj힄cia","command_time_left":"Pozosta흢y czas","command_arrive":"Czas dotarcia","error_no_units_enough":"Brak wystarczaj훳cej liczby jednostek do wys흢ania rozkazu!","error_not_own_village":"Wioska 탄r처d흢owa nie nale탉y do ciebie!","error_origin":"Nieprawid흢owa wioska 탄r처d흢owa!","error_target":"Nieprawid흢owa wioska cel!","error_no_units":"Nie wybrano jednostek!","error_invalid_date":"Nieprawid흢owy Czas","error_already_sent":"Ten rozkaz %d powinien zosta훶 wys흢any %d","error_no_map_selected_village":"Nie zaznaczono wioski na mapie.","error_remove_error":"B흢훳d usuwania rozkazu.","tab_add":"Dodaj rozkaz","tab_waiting":"Oczekuj훳ce","tab_logs":"Logi"}},"pt_br":{"command_queue":{"title":"CommandQueue","attack":"Ataque","support":"Apoio","relocate":"Transfer챗ncia","sent":"enviado","activated":"ativado","deactivated":"desativado","expired":"expirado","removed":"removido","added":"adicionado","general_clear":"Limpar registros","general_next_command":"Pr처ximo comando","add_basics":"Informa챌천es b찼sicas","add_origin":"Coordenadas da origem","add_selected":"Aldeia ativa","add_target":"Coordenadas do alvo","add_map_selected":"Aldeia selecionada no mapa","date_type_arrive":"Data de chegada","date_type_out":"Data de sa챠da","add_current_date":"Data/hora","add_current_date_plus":"Aumentar data em 100 milisegunds.","add_current_date_minus":"Reduzir data em 100 milisegunds.","add_travel_times":"Tempos de viagem","add_date":"Data","add_no_village":"selecione uma aldeia...","add_village_search":"Procurar aldeia...","add_clear":"Limpar campos","add_insert_preset":"Inserir predefini챌찾o","queue_waiting":"Comandos em espera","queue_none_added":"Nenhum comando adicionado.","queue_sent":"Comandos enviados","queue_none_sent":"Nenhum comando enviado.","queue_expired":"Comandos expirados","queue_none_expired":"Nenhum comando expirado.","queue_remove":"Remover comando da lista","queue_filters":"Filtro de comandos","filters_selected_village":"Mostrar apenas comandos com origem da aldeia selecionada","filters_barbarian_target":"Mostrar apenas comandos com aldeias b찼rbaras como alvo","filters_attack":"Mostrar ataques","filters_support":"Mostrar apoios","filters_relocate":"Mostrar transfer챗ncias","filters_text_match":"Filtrar por texto...","command_out":"Sa챠da na data","command_time_left":"Tempo restante","command_arrive":"Chegada na data","error_no_units_enough":"Sem unidades o sulficiente para enviar o comando!","error_not_own_village":"A aldeia de origem n찾o pertence a voc챗!","error_origin":"Aldeia de origem inv찼lida!","error_target":"Aldeia alvo inv찼lida!","error_no_units":"Nenhuma unidade especificada!","error_invalid_date":"Data inv찼lida","error_already_sent":"Esse %d deveria ter sa챠do %d","error_no_map_selected_village":"Nenhuma aldeia selecionada no mapa.","error_remove_error":"Erro ao remover comando.","tab_add":"Adicionar comando","tab_waiting":"Comandos em espera","tab_logs":"Registro de comandos"}},"ru_ru":{"command_queue":{"title":"��筠�筠畇� 克棘劇逵戟畇","attack":"逵�逵克逵","support":"極棘畇畇筠�菌克逵","relocate":"極筠�筠劇筠�筠戟龜筠","sent":"極棘�剋逵��","activated":"逵克�龜勻龜�棘勻逵��","deactivated":"棘�克剋��筠戟棘","expired":"龜��筠克�龜橘","removed":"�畇逵剋筠戟戟�橘","added":"畇棘閨逵勻剋筠戟棘","general_clear":"��龜��龜�� 菌��戟逵剋","general_next_command":"鬼剋筠畇���逵� 克棘劇逵戟畇逵","add_basics":"�棘畇�棘閨戟棘��龜","add_origin":"�棘棘�畇龜戟逵�� �棘�克龜 棘�極�逵勻剋筠戟龜�","add_selected":"龜筠克��逵� 畇筠�筠勻戟�","add_target":"�棘棘�畇龜戟逵�� �棘�克龜 戟逵鈞戟逵�筠戟龜�","add_map_selected":"��閨�逵戟戟逵� 畇筠�筠勻戟� 戟逵 克逵��筠","date_type_arrive":"�逵�逵 極�龜閨��龜�","date_type_out":"�逵�逵 棘�極�逵勻剋筠戟龜�","add_current_date":"龜筠克��逵� 畇逵�逵","add_current_date_plus":"叫勻筠剋龜�龜�� 畇逵�� 戟逵 100 劇龜剋剋龜�筠克�戟畇.","add_current_date_minus":"叫劇筠戟��龜�� 畇逵�� 戟逵 100 劇龜剋剋龜�筠克�戟畇.","add_travel_times":"��筠劇� 勻 極��龜","add_date":"�逵�逵/勻�筠劇�","add_no_village":"勻�閨�逵�� 畇筠�筠勻戟�...","add_village_search":"�棘龜�克 畇筠�筠勻戟龜...","add_clear":"��龜��龜�� 極棘剋�","add_insert_preset":"���逵勻龜�� �逵閨剋棘戟","queue_waiting":"�菌龜畇逵戟龜筠 克棘劇逵戟畇","queue_none_added":"�筠� 畇棘閨逵勻剋筠戟戟�� 克棘劇逵戟畇.","queue_sent":"��極�逵勻剋筠戟戟�筠 克棘劇逵戟畇�","queue_none_sent":"�筠� 棘�極�逵勻剋筠戟戟�� 克棘劇逵戟畇.","queue_expired":"��棘��棘�筠戟戟�筠 克棘劇逵戟畇�","queue_none_expired":"�筠� 鈞逵勻筠��筠戟戟�� 克棘劇逵戟畇.","queue_remove":"叫畇逵剋龜�� 克棘劇逵戟畇� 龜鈞 �極龜�克逵","queue_filters":"圭龜剋��� 克棘劇逵戟畇","filters_selected_village":"�棘克逵鈞逵�� �棘剋�克棘 克棘劇逵戟畇� 龜鈞 勻�閨�逵戟戟棘橘 畇筠�筠勻戟龜","filters_barbarian_target":"�棘克逵鈞逵�� �棘剋�克棘 克棘劇逵戟畇� � 畇筠�筠勻戟�劇龜 勻逵�勻逵�棘勻 勻 克逵�筠��勻筠 �筠剋龜","filters_attack":"�棘克逵鈞逵�� 逵�逵克龜","filters_support":"�棘克逵鈞逵�� 極棘畇克�筠極剋筠戟龜�","filters_relocate":"�棘克逵鈞逵�� 極筠�筠劇筠�筠戟龜�","filters_text_match":"圭龜剋��� 極棘 �筠克���...","command_out":"���棘畇","command_time_left":"��筠劇筠戟龜 棘��逵剋棘��","command_arrive":"��龜閨��龜筠","error_no_units_enough":"�筠畇棘��逵�棘�戟棘 勻棘橘�克 畇剋� 棘�極�逵勻克龜 克棘劇逵戟畇�!","error_not_own_village":"�筠�筠勻戟� 棘�極�逵勻剋筠戟龜� 極�龜戟逵畇剋筠菌龜� 戟筠 �逵劇!","error_origin":"�棘棘�畇龜戟逵�� 畇筠�筠勻戟龜 棘�極�逵勻剋筠戟龜� 戟筠勻筠�戟�!","error_target":"�棘棘�畇龜戟逵�� 畇筠�筠勻戟龜 戟逵鈞戟逵�筠戟龜� 戟筠勻筠�戟�!","error_no_units":"�筠 �克逵鈞逵戟� 勻棘橘�克逵!","error_invalid_date":"�筠勻筠�戟逵� 畇逵�逵","error_already_sent":"葵�棘 %d 畇棘剋菌戟棘 �橘�龜 %d","error_no_map_selected_village":"�筠� 勻�閨�逵戟戟棘橘 畇筠�筠勻戟龜 戟逵 克逵��筠.","error_remove_error":"��龜閨克逵 �畇逵剋筠戟龜� 克棘劇逵戟畇�.","tab_add":"�棘閨逵勻龜�� 克棘劇逵戟畇�","tab_waiting":"�棘劇逵戟畇� 勻 棘�筠�筠畇龜","tab_logs":"���戟逵剋 克棘劇逵戟畇"}}})
        commandQueue.init()
        commandQueueInterface()

        if (commandQueue.getWaitingCommands().length > 0) {
            commandQueue.start(true)
        }
    })
})

define('two/farmOverflow', [
    'two/Settings',
    'two/farmOverflow/types/errors',
    'two/farmOverflow/types/status',
    'two/farmOverflow/settings',
    'two/farmOverflow/settings/map',
    'two/farmOverflow/settings/updates',
    'two/farmOverflow/types/logs',
    'two/mapData',
    'two/utils',
    'helper/math',
    'helper/time',
    'queues/EventQueue',
    'conf/commandTypes',
    'conf/village',
    'struct/MapData',
    'Lockr'
], function (
    Settings,
    ERROR_TYPES,
    STATUS,
    SETTINGS,
    SETTINGS_MAP,
    UPDATES,
    LOG_TYPES,
    mapData,
    utils,
    math,
    timeHelper,
    eventQueue,
    COMMAND_TYPES,
    VILLAGE_CONFIG,
    $mapData,
    Lockr
) {
    let initialized = false
    let running = false
    let settings
    let farmers = []
    let logs = []
    let includedVillages = []
    let ignoredVillages = []
    let onlyVillages = []
    let selectedPresets = []
    let activeFarmer = false
    let sendingCommand = false
    let currentTarget = false
    let farmerIndex = 0
    let farmerCycle = []
    let farmerTimeoutId = null
    let targetTimeoutId = null
    let exceptionLogs
    let tempVillageReports = {}
    let $player
    let unitsData
    const VILLAGE_COMMAND_LIMIT = 50
    const MINIMUM_FARMER_CYCLE_INTERVAL = 5 * 1000
    const MINIMUM_ATTACK_INTERVAL = 1 * 500

    const STORAGE_KEYS = {
        LOGS: 'farm_overflow_logs',
        SETTINGS: 'farm_overflow_settings',
        EXCEPTION_LOGS: 'farm_overflow_exception_logs'
    }

    const villageFilters = {
        distance: function (target) {
            return !target.distance.between(
                settings.get(SETTINGS.MIN_DISTANCE),
                settings.get(SETTINGS.MAX_DISTANCE)
            )
        },
        ownPlayer: function (target) {
            return target.character_id === $player.getId()
        },
        included: function (target) {
            return target.character_id && !includedVillages.includes(target.id)
        },
        ignored: function (target) {
            return ignoredVillages.includes(target.id)
        },
        points: function (points) {
            return !points.between(
                settings.get(SETTINGS.MIN_POINTS),
                settings.get(SETTINGS.MAX_POINTS)
            )
        }
    }

    const targetFilters = [
        villageFilters.distance,
        villageFilters.ownPlayer,
        villageFilters.included,
        villageFilters.ignored
    ]

    const calcDistances = function (targets, origin) {
        return targets.map(function (target) {
            target.distance = math.actualDistance(origin, target)
            return target
        })
    }

    const filterTargets = function (targets) {
        return targets.filter(function (target) {
            return targetFilters.every(function (fn) {
                return !fn(target)
            })
        })
    }

    const sortTargets = function (targets) {
        return targets.sort(function (a, b) {
            return a.distance - b.distance
        })
    }

    const reloadTargets = function () {
        farmers.forEach(function (farmer) {
            farmer.loadTargets()
        })
    }

    const arrayUnique = function (array) {
        return array.sort().filter(function (item, pos, ary) {
            return !pos || item != ary[pos - 1]
        })
    }

    const skipStepInterval = function () {
        if (!running) {
            return
        }

        if (activeFarmer && targetTimeoutId) {
            clearTimeout(targetTimeoutId)
            targetTimeoutId = null
            activeFarmer.targetStep({
                delay: true
            })
        }

        if (farmerTimeoutId) {
            clearTimeout(farmerTimeoutId)
            farmerTimeoutId = null
            farmerIndex = 0
            farmOverflow.farmerStep()
        }
    }

    const updateIncludedVillage = function () {
        const groupsInclude = settings.get(SETTINGS.GROUP_INCLUDE)

        includedVillages = []

        groupsInclude.forEach(function (groupId) {
            let groupVillages = modelDataService.getGroupList().getGroupVillageIds(groupId)
            includedVillages = includedVillages.concat(groupVillages)
        })

        includedVillages = arrayUnique(includedVillages)
    }

    const updateIgnoredVillage = function () {
        const groupIgnored = settings.get(SETTINGS.GROUP_IGNORE)
        ignoredVillages = modelDataService.getGroupList().getGroupVillageIds(groupIgnored)
    }

    const updateOnlyVillage = function () {
        const groupsOnly = settings.get(SETTINGS.GROUP_ONLY)

        onlyVillages = []

        groupsOnly.forEach(function (groupId) {
            let groupVillages = modelDataService.getGroupList().getGroupVillageIds(groupId)
            groupVillages = groupVillages.filter(function (villageId) {
                return !!$player.getVillage(villageId)
            })

            onlyVillages = onlyVillages.concat(groupVillages)
        })

        onlyVillages = arrayUnique(onlyVillages)
    }

    const updateExceptionLogs = function () {
        const exceptionVillages = ignoredVillages.concat(includedVillages)
        let modified = false

        exceptionVillages.forEach(function (villageId) {
            if (!exceptionLogs.hasOwnProperty(villageId)) { 
                exceptionLogs[villageId] = {
                    time: timeHelper.gameTime(),
                    report: false
                }
                modified = true
            }
        })

        angular.forEach(exceptionLogs, function (time, villageId) {
            villageId = parseInt(villageId, 10)

            if (!exceptionVillages.includes(villageId)) {
                delete exceptionLogs[villageId]
                modified = true
            }
        })

        if (modified) {
            Lockr.set(STORAGE_KEYS.EXCEPTION_LOGS, exceptionLogs)
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_EXCEPTION_LOGS_UPDATED)
        }
    }

    const updateGroupVillages = function () {
        updateIncludedVillage()
        updateIgnoredVillage()
        updateOnlyVillage()
        updateExceptionLogs()

        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_EXCEPTION_VILLAGES_UPDATED)
    }

    const villageGroupLink = function (event, data) {
        const groupsInclude = settings.get(SETTINGS.GROUP_INCLUDE)
        const groupIgnore = settings.get(SETTINGS.GROUP_IGNORE)
        const groupsOnly = settings.get(SETTINGS.GROUP_ONLY)
        const isOwnVillage = $player.getVillage(data.village_id)
        let farmerListUpdated = false

        updateGroupVillages()

        if (groupIgnore === data.group_id) {
            if (isOwnVillage) {
                farmOverflow.removeById(data.village_id)
                farmerListUpdated = true
            } else {
                farmOverflow.removeTarget(data.village_id)

                addLog(LOG_TYPES.IGNORED_VILLAGE, {
                    villageId: data.village_id
                })
                addExceptionLog(data.village_id)
            }
        }

        if (groupsInclude.includes(data.group_id) && !isOwnVillage) {
            reloadTargets()

            addLog(LOG_TYPES.INCLUDED_VILLAGE, {
                villageId: data.village_id
            })
            addExceptionLog(data.village_id)
        }

        if (groupsOnly.includes(data.group_id) && isOwnVillage) {
            let farmer = farmOverflow.create(data.village_id)
            farmer.init().then(function () {
                if (running) {
                    farmer.start()
                }
            })

            farmerListUpdated = true
        }

        if (farmerListUpdated) {
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_FARMER_VILLAGES_UPDATED)
        }
    }

    const villageGroupUnlink = function (event, data) {
        const groupsInclude = settings.get(SETTINGS.GROUP_INCLUDE)
        const groupIgnore = settings.get(SETTINGS.GROUP_IGNORE)
        const groupsOnly = settings.get(SETTINGS.GROUP_ONLY)
        const isOwnVillage = $player.getVillage(data.village_id)
        let farmerListUpdated = false

        updateGroupVillages()

        if (groupIgnore === data.group_id) {
            if (isOwnVillage) {
                let farmer = farmOverflow.create(data.village_id)
                farmer.init().then(function () {
                    if (running) {
                        farmer.start()
                    }
                })

                farmerListUpdated = true
            } else {
                reloadTargets()

                addLog(LOG_TYPES.IGNORED_VILLAGE_REMOVED, {
                    villageId: data.village_id
                })
            }
        }

        if (groupsInclude.includes(data.group_id) && !isOwnVillage) {
            reloadTargets()

            addLog(LOG_TYPES.INCLUDED_VILLAGE_REMOVED, {
                villageId: data.village_id
            })
        }

        if (groupsOnly.includes(data.group_id) && isOwnVillage) {
            farmOverflow.removeById(data.village_id)
            farmerListUpdated = true
        }

        if (farmerListUpdated) {
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_FARMER_VILLAGES_UPDATED)
        }
    }

    const removedGroupListener = function () {
        updateGroupVillages()

        farmOverflow.flush()
        reloadTargets()
        farmOverflow.createAll()
    }

    const updatePresets = function () {
        const processPresets = function () {
            selectedPresets = []
            const playerPresets = modelDataService.getPresetList().getPresets()
            const activePresets = settings.get(SETTINGS.PRESETS)

            activePresets.forEach(function (presetId) {
                if (!playerPresets.hasOwnProperty(presetId)) {
                    return
                }

                preset = playerPresets[presetId]
                preset.load = getPresetHaul(preset)
                preset.travelTime = getPresetTimeTravel(preset, false)
                selectedPresets.push(preset)
            })

            selectedPresets = selectedPresets.sort(function (a, b) {
                return a.travelTime - b.travelTime || b.load - a.load
            })
        }

        if (modelDataService.getPresetList().isLoaded()) {
            processPresets()
        } else {
            socketService.emit(routeProvider.GET_PRESETS, {}, processPresets)
        }
    }

    const ignoreVillage = function (villageId) {
        const groupIgnore = settings.get(SETTINGS.GROUP_IGNORE)

        if (!groupIgnore) {
            return false
        }

        socketService.emit(routeProvider.GROUPS_LINK_VILLAGE, {
            group_id: groupIgnore,
            village_id: villageId
        })

        return true
    }

    const presetListener = function () {
        updatePresets()

        if (!selectedPresets.length) {
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_STOP, {
                reason: ERROR_TYPES.NO_PRESETS
            })

            if (running) {
                farmOverflow.stop()
            }
        }
    }

    const reportListener = function (event, data) {
        if (!settings.get(SETTINGS.IGNORE_ON_LOSS) || !settings.get(SETTINGS.GROUP_IGNORE)) {
            return
        }

        if (!running || data.type !== COMMAND_TYPES.TYPES.ATTACK) {
            return
        }

        // 1 = nocasualties
        // 2 = casualties
        // 3 = defeat
        if (data.result !== 1 && farmOverflow.isTarget(data.target_village_id)) {
            tempVillageReports[data.target_village_id] = {
                haul: data.haul,
                id: data.id,
                result: data.result,
                title: data.title
            }

            ignoreVillage(data.target_village_id)
        }
    }

    const commandSentListener = function (event, data) {
        if (!activeFarmer || !currentTarget) {
            return
        }

        if (data.origin.id !== activeFarmer.getVillage().getId()) {
            return
        }

        if (data.target.id !== currentTarget.id) {
            return
        }

        if (data.direction === 'forward' && data.type === COMMAND_TYPES.TYPES.ATTACK) {
            activeFarmer.commandSent(data)
        }
    }

    const commandErrorListener = function (event, data) {
        if (!activeFarmer || !sendingCommand || !currentTarget) {
            return
        }

        if (data.cause === routeProvider.SEND_PRESET.type) {
            activeFarmer.commandError(data)
        }
    }

    const assignPresets = function (villageId, presetIds, callback) {
        socketService.emit(routeProvider.ASSIGN_PRESETS, {
            village_id: villageId,
            preset_ids: presetIds
        }, callback)
    }

    const getPresetHaul = function (preset) {
        let haul = 0

        angular.forEach(preset.units, function (unitAmount, unitName) {
            if (unitAmount) {
                haul += unitsData[unitName].load * unitAmount
            }
        })

        return haul
    }

    const getPresetTimeTravel = function (preset, barbarian) {
        return armyService.calculateTravelTime(preset, {
            barbarian: barbarian,
            officers: false
        })
    }

    const checkPresetTime = function (preset, village, target) {
        const limitTime = settings.get(SETTINGS.MAX_TRAVEL_TIME) * 60
        const position = village.getPosition()
        const distance = math.actualDistance(position, target)
        const travelTime = getPresetTimeTravel(preset, !target.character_id)
        const totalTravelTime = armyService.getTravelTimeForDistance(preset, travelTime, distance, COMMAND_TYPES.TYPES.ATTACK)

        return limitTime > totalTravelTime
    }

    const checkFullStorage = function(village) {
        if (!village.isReady()) {
            return false
        }

        const resources = village.getResources()
        const computed = resources.getComputed()
        const maxStorage = resources.getMaxStorage()

        return ['wood', 'clay', 'iron'].every(function (type) {
            return computed[type].currentStock === maxStorage
        })
    }

    const addExceptionLog = function (villageId) {
        exceptionLogs[villageId] = {
            time: timeHelper.gameTime(),
            report: tempVillageReports[villageId] || false
        }

        delete tempVillageReports[villageId]

        Lockr.set(STORAGE_KEYS.EXCEPTION_LOGS, exceptionLogs)
        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_EXCEPTION_LOGS_UPDATED)
    }

    const addLog = function(type, _data) {
        if (typeof type !== 'string') {
            return false
        }

        let data = angular.isObject(_data) ? _data : {}

        data.time = timeHelper.gameTime()
        data.type = type

        logs.unshift(data)
        trimAndSaveLogs()

        return true
    }

    const trimAndSaveLogs = function () {
        const limit = settings.get(SETTINGS.LOGS_LIMIT)

        if (logs.length > limit) {
            logs.splice(logs.length - limit, logs.length)
        }

        Lockr.set(STORAGE_KEYS.LOGS, logs)
        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_LOGS_UPDATED)
    }

    const Farmer = function (villageId) {
        let self = this
        let village = $player.getVillage(villageId)
        let index = 0
        let running = false
        let initialized = false
        let ready = false
        let targets = false
        let cycleEndHandler = noop
        let loadPromises
        let status = STATUS.WAITING_CYCLE

        if (!village) {
            throw new Error(`new Farmer -> Village ${villageId} doesn't exist.`)
        }

        self.init = function () {
            loadPromises = []
            initialized = true

            if (!self.isReady()) {
                loadPromises.push(new Promise(function(resolve) {
                    if (self.isReady()) {
                        return resolve()
                    }

                    villageService.ensureVillageDataLoaded(village.getId(), resolve)
                }))

                loadPromises.push(new Promise(function (resolve) {
                    if (self.isReady()) {
                        return resolve()
                    }

                    self.loadTargets(function () {
                        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_READY, {
                            villageId: villageId
                        })
                        resolve()
                    })
                }))
            }

            return Promise.all(loadPromises).then(function() {
                ready = true
            })
        }

        self.start = function () {
            if (running) {
                return false
            }

            if (!ready) {
                eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_ERROR_NOT_READY, {
                    villageId: villageId
                })
                return false
            }

            if (!targets.length) {
                eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_ERROR_NO_TARGETS, {
                    villageId: villageId
                })
                return false
            }

            activeFarmer = this
            running = true
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_START, {
                villageId: villageId
            })

            self.targetStep({
                delay: false
            })

            return true
        }

        self.stop = function (reason) {
            running = false
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_STOP, {
                villageId: villageId,
                reason: reason
            })

            if (reason !== ERROR_TYPES.USER_STOP) {
                cycleEndHandler(reason)
            }

            clearTimeout(targetTimeoutId)
            cycleEndHandler = noop
        }

        self.targetStep = function (_options) {
            const options = _options || {}
            let preset
            let delayTime = 0
            let target
            const commandList = village.getCommandListModel()
            const villageCommands = commandList.getOutgoingCommands(true, true)
            let checkedLocalCommands = false

            const checkCommandLimit = function () {
                return new Promise(function (resolve, reject) {
                    const limit = VILLAGE_COMMAND_LIMIT - settings.get(SETTINGS.PRESERVE_COMMAND_SLOTS)

                    if (villageCommands.length >= limit) {
                        return reject(STATUS.COMMAND_LIMIT)
                    }

                    resolve()
                })
            }

            const checkStorage = function () {
                return new Promise(function (resolve, reject) {
                    if (settings.get(SETTINGS.IGNORE_FULL_STORAGE) && checkFullStorage(village)) {
                        return reject(STATUS.FULL_STORAGE)
                    }

                    resolve()
                })
            }

            const checkTargets = function () {
                return new Promise(function (resolve, reject) {
                    if (!targets.length) {
                        reject(STATUS.NO_TARGETS)
                    } else if (index > targets.length || !targets[index]) {
                        reject(STATUS.TARGET_CYCLE_END)
                    } else {
                        resolve()
                    }
                })
            }

            const checkVillagePresets = function () {
                return new Promise(function (resolve) {
                    const neededPresets = genPresetList()

                    if (neededPresets) {
                        assignPresets(village.getId(), neededPresets, resolve)
                    } else {
                        resolve()
                    }
                })
            }

            const checkPreset = function () {
                return new Promise(function (resolve, reject) {
                    target = getTarget()

                    if (!target) {
                        return reject(STATUS.UNKNOWN)
                    }

                    preset = getPreset(target)

                    if (typeof preset === 'string') {
                        return reject(preset)
                    }

                    resolve()
                })
            }

            const checkTarget = function () {
                return new Promise(function (resolve, reject) {
                    socketService.emit(routeProvider.GET_ATTACKING_FACTOR, {
                        target_id: target.id
                    }, function(data) {
                        // abandoned village conquered by some noob.
                        if (target.character_id === null && data.owner_id !== null && !includedVillages.includes(target.id)) {
                            reject(STATUS.ABANDONED_CONQUERED)
                        } else if (target.attack_protection) {
                            reject(STATUS.PROTECTED_VILLAGE)
                        } else {
                            resolve()
                        }
                    })
                })
            }

            const checkLocalCommands = function () {
                return new Promise(function (resolve, reject) {
                    let otherAttacking = false

                    const multipleFarmers = settings.get(SETTINGS.TARGET_MULTIPLE_FARMERS)
                    const singleAttack = settings.get(SETTINGS.TARGET_SINGLE_ATTACK)
                    const playerVillages = $player.getVillageList()
                    const allVillagesLoaded = playerVillages.every(function (anotherVillage) {
                        return anotherVillage.isReady(VILLAGE_CONFIG.READY_STATES.OWN_COMMANDS)
                    })

                    if (allVillagesLoaded) {
                        otherAttacking = playerVillages.some(function (anotherVillage) {
                            if (anotherVillage.getId() === village.getId()) {
                                return false
                            }

                            const anotherVillageCommands = anotherVillage.getCommandListModel().getOutgoingCommands(true, true)

                            return anotherVillageCommands.some(function (command) {
                                return command.targetVillageId === target.id && command.data.direction === 'forward'
                            })
                        })
                    }

                    const attacking = villageCommands.some(function (command) {
                        return command.data.target.id === target.id && command.data.direction === 'forward'
                    })

                    if (isTargetBusy(attacking, otherAttacking, allVillagesLoaded)) {
                        return reject(STATUS.BUSY_TARGET)
                    }

                    if (allVillagesLoaded) {
                        checkedLocalCommands = true
                    }

                    resolve()
                })
            }

            const checkTargetPoints = function () {
                return new Promise(function (resolve, reject) {
                    $mapData.getTownAtAsync(target.x, target.y, function (data) {
                        if (villageFilters.points(data.points)) {
                            return reject(STATUS.NOT_ALLOWED_POINTS)
                        }

                        resolve()
                    })
                })
            }

            const checkLoadedCommands = function () {
                if (checkedLocalCommands) {
                    return true
                }

                return new Promise(function (resolve, reject) {
                    socketService.emit(routeProvider.MAP_GET_VILLAGE_DETAILS, {
                        my_village_id: villageId,
                        village_id: target.id,
                        num_reports: 0
                    }, function (data) {
                        const targetCommands = data.commands.own.filter(function (command) {
                            return command.type === COMMAND_TYPES.TYPES.ATTACK && command.direction === 'forward'
                        })
                        const multipleFarmers = settings.get(SETTINGS.TARGET_MULTIPLE_FARMERS)
                        const singleAttack = settings.get(SETTINGS.TARGET_SINGLE_ATTACK)
                        const otherAttacking = targetCommands.some(function (command) {
                            return command.start_village_id !== villageId
                        })
                        const attacking = targetCommands.some(function (command) {
                            return command.start_village_id === villageId
                        })

                        if (isTargetBusy(attacking, otherAttacking, true)) {
                            return reject(STATUS.BUSY_TARGET)
                        }

                        resolve()
                    })
                })
            }

            const prepareAttack = function () {
                if (options.delay) {
                    delayTime = utils.randomSeconds(settings.get(SETTINGS.ATTACK_INTERVAL))
                    delayTime = MINIMUM_ATTACK_INTERVAL + (delayTime * 1000)
                }

                self.setStatus(STATUS.ATTACKING)

                targetTimeoutId = setTimeout(function() {
                    attackTarget(target, preset)
                }, delayTime)
            }

            const onError = function (error) {
                console.log(error)

                eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_INSTANCE_STEP_ERROR, {
                    villageId: villageId,
                    error: error
                })

                index++

                switch (error) {
                case STATUS.TIME_LIMIT:
                case STATUS.BUSY_TARGET:
                case STATUS.ABANDONED_CONQUERED:
                case STATUS.PROTECTED_VILLAGE:
                    self.setStatus(error)
                    self.targetStep(options)
                    break

                case STATUS.NOT_ALLOWED_POINTS:
                    self.setStatus(error)
                    farmOverflow.removeTarget(target.id)
                    self.targetStep(options)
                    break

                case STATUS.NO_UNITS:
                case STATUS.NO_TARGETS:
                case STATUS.FULL_STORAGE:
                case STATUS.COMMAND_LIMIT:
                    self.setStatus(error)
                    self.stop(error)
                    break

                case STATUS.TARGET_CYCLE_END:
                    self.setStatus(error)
                    self.stop(error)
                    index = 0
                    break

                default:
                    self.setStatus(STATUS.UNKNOWN)
                    self.stop(STATUS.UNKNOWN)
                    break
                }
            }

            checkCommandLimit()
                .then(checkStorage)
                .then(checkTargets)
                .then(checkVillagePresets)
                .then(checkPreset)
                .then(checkTarget)
                .then(checkLocalCommands)
                .then(checkTargetPoints)
                .then(checkLoadedCommands)
                .then(prepareAttack)
                .catch(onError)
        }

        self.setStatus = function (newStatus) {
            status = newStatus
        }

        self.getStatus = function () {
            return status || ''
        }

        self.commandSent = function (data) {
            sendingCommand = false
            currentTarget = false

            self.targetStep({
                delay: true
            })
        }

        self.commandError = function (data) {
            sendingCommand = false
            currentTarget = false

            self.stop(STATUS.COMMAND_ERROR)
        }

        self.onceCycleEnd = function (handler) {
            cycleEndHandler = handler
        }

        self.loadTargets = function (_callback) {
            const pos = village.getPosition()

            mapData.load(pos, function (loadedTargets) {
                targets = calcDistances(loadedTargets, pos)
                targets = filterTargets(targets, pos)
                targets = sortTargets(targets)

                if (typeof _callback === 'function') {
                    _callback(targets)
                }
            })
        }

        self.getTargets = function () {
            return targets
        }

        self.getIndex = function () {
            return index
        }

        self.getVillage = function () {
            return village
        }

        self.isRunning = function () {
            return running
        }

        self.isReady = function () {
            return initialized && ready
        }

        self.removeTarget = function (targetId) {
            if (typeof targetId !== 'number' || !targets) {
                return false
            }

            targets = targets.filter(function (target) {
                return target.id !== targetId
            })

            return true
        }

        // private functions

        const genPresetList = function () {
            const villagePresets = modelDataService.getPresetList().getPresetsByVillageId(village.getId())
            let needAssign = false
            let which = []

            selectedPresets.forEach(function (preset) {
                if (!villagePresets.hasOwnProperty(preset.id)) {
                    needAssign = true
                    which.push(preset.id)
                }
            })

            if (needAssign) {
                for (let id in villagePresets) {
                    which.push(id)
                }

                return which
            }

            return false
        }

        const isTargetBusy = function (attacking, otherAttacking, allVillagesLoaded) {
            const multipleFarmers = settings.get(SETTINGS.TARGET_MULTIPLE_FARMERS)
            const singleAttack = settings.get(SETTINGS.TARGET_SINGLE_ATTACK)

            if (multipleFarmers && allVillagesLoaded) {
                if (singleAttack && attacking) {
                    return true
                }
            } else if (singleAttack) {
                if (attacking || otherAttacking) {
                    return true
                }
            } else if (otherAttacking) {
                return true
            }

            return false
        }

        const attackTarget = function (target, preset) {
            if (!running) {
                return false
            }

            sendingCommand = true
            currentTarget = target
            index++

            socketService.emit(routeProvider.SEND_PRESET, {
                start_village: village.getId(),
                target_village: target.id,
                army_preset_id: preset.id,
                type: COMMAND_TYPES.TYPES.ATTACK
            })
        }

        const getTarget = function () {
            return targets[index]
        }

        const getPreset = function (target) {
            const units = village.getUnitInfo().getUnits()

            for (let i = 0; i < selectedPresets.length; i++) {
                let preset = selectedPresets[i]
                let avail = true

                for (let unit in preset.units) {
                    if (!preset.units[unit]) {
                        continue
                    }

                    if (units[unit].in_town < preset.units[unit]) {
                        avail = false
                    }
                }

                if (avail) {
                    if (checkPresetTime(preset, village, target)) {
                        return preset
                    } else {
                        return STATUS.TIME_LIMIT
                    }
                }
            }

            return STATUS.NO_UNITS
        }
    }

    let farmOverflow = {}

    farmOverflow.init = function () {
        initialized = true
        logs = Lockr.get(STORAGE_KEYS.LOGS, [])
        exceptionLogs = Lockr.get(STORAGE_KEYS.EXCEPTION_LOGS, {})
        $player = modelDataService.getSelectedCharacter()
        unitsData = modelDataService.getGameData().getUnitsObject()

        settings = new Settings({
            settingsMap: SETTINGS_MAP,
            storageKey: STORAGE_KEYS.SETTINGS
        })

        settings.onChange(function (changes, updates) {
            if (updates[UPDATES.PRESET]) {
                updatePresets()
            }

            if (updates[UPDATES.GROUPS]) {
                updateGroupVillages()
            }

            if (updates[UPDATES.TARGETS]) {
                reloadTargets()
            }

            if (updates[UPDATES.VILLAGES]) {
                farmOverflow.flush()
                farmOverflow.createAll()
            }

            if (updates[UPDATES.LOGS]) {
                trimAndSaveLogs()
            }

            if (updates[UPDATES.INTERVAL_TIMERS]) {
                skipStepInterval()
            }
        })

        updateGroupVillages()
        updatePresets()
        farmOverflow.createAll()

        $rootScope.$on(eventTypeProvider.ARMY_PRESET_UPDATE, presetListener)
        $rootScope.$on(eventTypeProvider.ARMY_PRESET_DELETED, presetListener)
        $rootScope.$on(eventTypeProvider.GROUPS_VILLAGE_LINKED, villageGroupLink)
        $rootScope.$on(eventTypeProvider.GROUPS_VILLAGE_UNLINKED, villageGroupUnlink)
        $rootScope.$on(eventTypeProvider.GROUPS_DESTROYED, removedGroupListener)
        $rootScope.$on(eventTypeProvider.COMMAND_SENT, commandSentListener)
        $rootScope.$on(eventTypeProvider.MESSAGE_ERROR, commandErrorListener)
        $rootScope.$on(eventTypeProvider.REPORT_NEW, reportListener)
    }

    farmOverflow.start = function () {
        let readyFarmers

        if (running) {
            return false
        }

        if (!selectedPresets.length) {
            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_STOP, {
                reason: ERROR_TYPES.NO_PRESETS
            })

            return false
        }

        running = true
        readyFarmers = []

        farmers.forEach(function (farmer) {
            readyFarmers.push(new Promise(function (resolve) {
                farmer.init().then(resolve)
            }))
        })

        if (!readyFarmers.length) {
            running = false
            return false
        }

        Promise.all(readyFarmers).then(function () {
            farmOverflow.farmerStep()
        })

        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_START)

        addLog(LOG_TYPES.FARM_START)
    }

    farmOverflow.stop = function (reason) {
        reason = reason || ERROR_TYPES.USER_STOP

        if (activeFarmer) {
            activeFarmer.stop(reason)
        }

        running = false

        clearTimeout(farmerTimeoutId)
        farmerTimeoutId = null

        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_STOP, {
            reason: reason
        })

        if (reason === ERROR_TYPES.USER_STOP) {
            addLog(LOG_TYPES.FARM_STOP)
        }
    }

    farmOverflow.createAll = function () {
        angular.forEach($player.getVillages(), function (village, villageId) {
            farmOverflow.create(villageId)
        })

        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_FARMER_VILLAGES_UPDATED)
    }

    farmOverflow.create = function (villageId) {
        const groupsOnly = settings.get(SETTINGS.GROUP_ONLY)

        villageId = parseInt(villageId, 10)

        if (groupsOnly.length && !onlyVillages.includes(villageId)) {
            return false
        }

        if (ignoredVillages.includes(villageId)) {
            return false
        }

        if (!farmOverflow.getById(villageId)) {
            farmers.push(new Farmer(villageId))
        }

        return farmOverflow.getById(villageId)
    }

    farmOverflow.flush = function () {
        const groupsOnly = settings.get(SETTINGS.GROUP_ONLY)
        let removeIds = []

        farmers.forEach(function (farmer) {
            let villageId = farmer.getVillage().getId()

            if (groupsOnly.length && !onlyVillages.includes(villageId)) {
                removeIds.push(villageId)
            } else if (ignoredVillages.includes(villageId)) {
                removeIds.push(villageId)
            }
        })

        if (removeIds.length) {
            removeIds.forEach(function (removeId) {
                farmOverflow.removeById(removeId)
            })

            eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_FARMER_VILLAGES_UPDATED)
        }
    }

    farmOverflow.getById = function (farmerId) {
        for (let i = 0; i < farmers.length; i++) {
            if (farmers[i].getVillage().getId() === farmerId) {
                return farmers[i]
            }
        }

        return false
    }

    farmOverflow.getAll = function () {
        return farmers
    }

    farmOverflow.removeById = function (farmerId) {
        for (let i = 0; i < farmers.length; i++) {
            if (farmers[i].getVillage().getId() === farmerId) {
                farmers[i].stop()
                farmers.splice(i, i + 1)

                return true
            }
        }

        return false
    }

    farmOverflow.getOne = function () {
        if (!farmers.length) {
            return false
        }

        if (farmerIndex >= farmers.length) {
            farmerIndex = 0
            return false
        }

        return farmers[farmerIndex]
    }

    farmOverflow.farmerStep = function () {
        activeFarmer = farmOverflow.getOne()

        if (!activeFarmer) {
            let interval = settings.get(SETTINGS.FARMER_CYCLE_INTERVAL) * 60 * 1000
            interval += MINIMUM_FARMER_CYCLE_INTERVAL

            farmerTimeoutId = setTimeout(function() {
                farmerTimeoutId = null
                farmerIndex = 0
                farmOverflow.farmerStep()
            }, interval)

            return
        }

        activeFarmer.onceCycleEnd(function () {
            farmerIndex++
            farmOverflow.farmerStep()
        })

        activeFarmer.start()
    }

    farmOverflow.isTarget = function (targetId) {
        for (let i = 0; i < farmers.length; i++) {
            let farmer = farmers[i]
            let targets = farmer.getTargets()

            for (let j = 0; j < targets.length; j++) {
                let target = targets[j]

                if (target.id === targetId) {
                    return true
                }
            }
        }

        return false
    }

    farmOverflow.removeTarget = function (targetId) {
        farmers.forEach(function (farmer) {
            farmer.removeTarget(targetId)
        })
    }

    farmOverflow.getSettings = function () {
        return settings
    }

    farmOverflow.getExceptionVillages = function () {
        return {
            included: includedVillages,
            ignored: ignoredVillages
        }
    }

    farmOverflow.getExceptionLogs = function () {
        return exceptionLogs
    }

    farmOverflow.isInitialized = function () {
        return initialized
    }

    farmOverflow.isRunning = function () {
        return running
    }

    farmOverflow.getLogs = function () {
        return logs
    }

    farmOverflow.clearLogs = function () {
        logs = []
        Lockr.set(STORAGE_KEYS.LOGS, logs)
        eventQueue.trigger(eventTypeProvider.FARM_OVERFLOW_LOGS_UPDATED)

        return logs
    }

    return farmOverflow
})

define('two/farmOverflow/events', [], function () {
    angular.extend(eventTypeProvider, {
        FARM_OVERFLOW_START: 'farm_overflow_start',
        FARM_OVERFLOW_STOP: 'farm_overflow_stop',
        FARM_OVERFLOW_INSTANCE_READY: 'farm_overflow_instance_ready',
        FARM_OVERFLOW_INSTANCE_START: 'farm_overflow_instance_start',
        FARM_OVERFLOW_INSTANCE_STOP: 'farm_overflow_instance_stop',
        FARM_OVERFLOW_INSTANCE_ERROR_NO_TARGETS: 'farm_overflow_instance_error_no_targets',
        FARM_OVERFLOW_INSTANCE_ERROR_NOT_READY: 'farm_overflow_instance_error_not_ready',
        FARM_OVERFLOW_INSTANCE_STEP_ERROR: 'farm_overflow_instance_command_error',
        FARM_OVERFLOW_PRESETS_LOADED: 'farm_overflow_presets_loaded',
        FARM_OVERFLOW_LOGS_UPDATED: 'farm_overflow_log_updated',
        FARM_OVERFLOW_COMMAND_SENT: 'farm_overflow_command_sent',
        FARM_OVERFLOW_IGNORED_TARGET: 'farm_overflow_ignored_target',
        FARM_OVERFLOW_VILLAGE_IGNORED: 'farm_overflow_village_ignored',
        FARM_OVERFLOW_EXCEPTION_VILLAGES_UPDATED: 'farm_overflow_exception_villages_updated',
        FARM_OVERFLOW_FARMER_VILLAGES_UPDATED: 'farm_overflow_farmer_villages_updated',
        FARM_OVERFLOW_REPORTS_UPDATED: 'farm_overflow_reports_updated',
        FARM_OVERFLOW_EXCEPTION_LOGS_UPDATED: 'farm_overflow_exception_logs_updated'
    })
})

define('two/farmOverflow/ui', [
    'two/ui',
    'two/farmOverflow',
    'two/farmOverflow/types/errors',
    'two/farmOverflow/types/logs',
    'two/farmOverflow/settings',
    'two/Settings',
    'two/EventScope',
    'two/utils',
    'queues/EventQueue',
    'struct/MapData',
    'helper/time',
], function (
    interfaceOverflow,
    farmOverflow,
    ERROR_TYPES,
    LOG_TYPES,
    SETTINGS,
    Settings,
    EventScope,
    utils,
    eventQueue,
    $mapData,
    timeHelper
) {
    let $scope
    let settings
    let presetList = modelDataService.getPresetList()
    let groupList = modelDataService.getGroupList()
    let $button
    let villagesInfo = {}
    let villagesLabel = {}

    const TAB_TYPES = {
        SETTINGS: 'settings',
        VILLAGES: 'villages',
        LOGS: 'logs'
    }

    const updateVisibleLogs = function () {
        const offset = $scope.pagination.offset
        const limit = $scope.pagination.limit

        $scope.visibleLogs = $scope.logs.slice(offset, offset + limit)
        $scope.pagination.count = $scope.logs.length

        $scope.visibleLogs.forEach(function (log) {
            if (log.villageId) {
                loadVillageInfo(log.villageId)
            }
        })
    }

    const loadVillageInfo = function (villageId) {
        if (villagesInfo[villageId]) {
            return villagesInfo[villageId]
        }

        villagesInfo[villageId] = true
        villagesLabel[villageId] = 'LOADING...'

        socketService.emit(routeProvider.MAP_GET_VILLAGE_DETAILS, {
            my_village_id: modelDataService.getSelectedVillage().getId(),
            village_id: villageId,
            num_reports: 1
        }, function (data) {
            villagesInfo[villageId] = {
                x: data.village_x,
                y: data.village_y,
                name: data.village_name,
                last_report: data.last_reports[0]
            }

            villagesLabel[villageId] = `${data.village_name} (${data.village_x}|${data.village_y})`
        })
    }

    const loadExceptionsInfo = function () {
        $scope.exceptionVillages.included.forEach(function (villageId) {
            loadVillageInfo(villageId)
        })
        $scope.exceptionVillages.ignored.forEach(function (villageId) {
            loadVillageInfo(villageId)
        })
    }

    const switchFarm = function () {
        if (farmOverflow.isRunning()) {
            farmOverflow.stop()
        } else {
            farmOverflow.start()
        }
    }

    const selectTab = function (tabType) {
        $scope.selectedTab = tabType
    }

    const saveSettings = function () {
        settings.setAll(settings.decode($scope.settings))

        $rootScope.$broadcast(eventTypeProvider.MESSAGE_SUCCESS, {
            message: $filter('i18n')('settings_saved', $rootScope.loc.ale, 'farm_overflow')
        })
    }

    const removeIgnored = function (villageId) {
        const groupIgnore = settings.get(SETTINGS.GROUP_IGNORE)
        const groupVillages = modelDataService.getGroupList().getGroupVillageIds(groupIgnore)

        if (!groupVillages.includes(villageId)) {
            return false
        }

        socketService.emit(routeProvider.GROUPS_UNLINK_VILLAGE, {
            group_id: groupIgnore,
            village_id: villageId
        })
    }

    const removeIncluded = function (villageId) {
        const groupsInclude = settings.get(SETTINGS.GROUP_INCLUDE)

        groupsInclude.forEach(function (groupId) {
            let groupVillages = modelDataService.getGroupList().getGroupVillageIds(groupId)

            if (groupVillages.includes(villageId)) {
                socketService.emit(routeProvider.GROUPS_UNLINK_VILLAGE, {
                    group_id: groupId,
                    village_id: villageId
                })
            }
        })
    }

    const eventHandlers = {
        updatePresets: function () {
            $scope.presets = Settings.encodeList(presetList.getPresets(), {
                disabled: false,
                type: 'presets'
            })
        },
        updateGroups: function () {
            $scope.groups = Settings.encodeList(groupList.getGroups(), {
                disabled: false,
                type: 'groups'
            })

            $scope.groupsWithDisabled = Settings.encodeList(groupList.getGroups(), {
                disabled: true,
                type: 'groups'
            })
        },
        start: function () {
            $scope.running = true

            $rootScope.$broadcast(eventTypeProvider.MESSAGE_SUCCESS, {
                message: $filter('i18n')('farm_started', $rootScope.loc.ale, 'farm_overflow')
            })
        },
        stop: function (event, data) {
            $scope.running = false

            switch (data.reason) {
            case ERROR_TYPES.NO_PRESETS:
                $rootScope.$broadcast(eventTypeProvider.MESSAGE_SUCCESS, {
                    message: $filter('i18n')('no_preset', $rootScope.loc.ale, 'farm_overflow')
                })
                break

            default:
            case ERROR_TYPES.USER_STOP:
                $rootScope.$broadcast(eventTypeProvider.MESSAGE_SUCCESS, {
                    message: $filter('i18n')('farm_stopped', $rootScope.loc.ale, 'farm_overflow')
                })
                break
            }
        },
        updateLogs: function () {
            $scope.logs = angular.copy(farmOverflow.getLogs())
            updateVisibleLogs()

            if (!$scope.logs.length) {
                $rootScope.$broadcast(eventTypeProvider.MESSAGE_SUCCESS, {
                    message: $filter('i18n')('reseted_logs', $rootScope.loc.ale, 'farm_overflow')
                })
            }
        },
        updateFarmerVillages: function () {
            $scope.farmers = farmOverflow.getAll()
        },
        updateExceptionVillages: function () {
            $scope.exceptionVillages = farmOverflow.getExceptionVillages()
            loadExceptionsInfo()
        },
        updateExceptionLogs: function () {
            $scope.exceptionLogs = farmOverflow.getExceptionLogs()
        }
    }

    const init = function () {
        settings = farmOverflow.getSettings()
        $button = interfaceOverflow.addMenuButton('Farmer', 10)

        $button.addEventListener('click', function () {
            buildWindow()
        })

        eventQueue.register(eventTypeProvider.FARM_OVERFLOW_START, function () {
            $button.classList.remove('btn-green')
            $button.classList.add('btn-red')
        })

        eventQueue.register(eventTypeProvider.FARM_OVERFLOW_STOP, function () {
            $button.classList.remove('btn-red')
            $button.classList.add('btn-green')
        })

        interfaceOverflow.addTemplate('twoverflow_farm_overflow_window', `<div id=\"two-farmoverflow\" class=\"win-content two-window\">
																																																																																																																																																																																																																																																																																																																																																																															<header class=\"win-head\">
																																																																																																																																																																																																																																																																																																																																																																																<h2>FarmOverflow</h2>
																																																																																																																																																																																																																																																																																																																																																																																<ul class=\"list-btn\">
																																																																																																																																																																																																																																																																																																																																																																																	<li>
																																																																																																																																																																																																																																																																																																																																																																																		<a href=\"#\" class=\"size-34x34 btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																																																																																																																																																																																																																																																																																																	</ul>
																																																																																																																																																																																																																																																																																																																																																																																</header>
																																																																																																																																																																																																																																																																																																																																																																																<div class=\"win-main\" scrollbar=\"\">
																																																																																																																																																																																																																																																																																																																																																																																	<div class=\"tabs tabs-bg\">
																																																																																																																																																																																																																																																																																																																																																																																		<div class=\"tabs-three-col\">
																																																																																																																																																																																																																																																																																																																																																																																			<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.SETTINGS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.SETTINGS}\">
																																																																																																																																																																																																																																																																																																																																																																																				<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																					<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.SETTINGS}\">
																																																																																																																																																																																																																																																																																																																																																																																						<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.SETTINGS}\">{{ TAB_TYPES.SETTINGS | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																																																			<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.VILLAGES)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.VILLAGES}\">
																																																																																																																																																																																																																																																																																																																																																																																				<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																					<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.VILLAGES}\">
																																																																																																																																																																																																																																																																																																																																																																																						<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.VILLAGES}\">{{ TAB_TYPES.VILLAGES | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																																																			<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.LOGS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.LOGS}\">
																																																																																																																																																																																																																																																																																																																																																																																				<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																					<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.LOGS}\">
																																																																																																																																																																																																																																																																																																																																																																																						<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.LOGS}\">{{ TAB_TYPES.LOGS | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																																																																																																																																																	</div>
																																																																																																																																																																																																																																																																																																																																																																																	<div class=\"box-paper footer\">
																																																																																																																																																																																																																																																																																																																																																																																		<div class=\"scroll-wrap\">
																																																																																																																																																																																																																																																																																																																																																																																			<div class=\"settings\" ng-show=\"selectedTab === TAB_TYPES.SETTINGS\">
																																																																																																																																																																																																																																																																																																																																																																																				<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																																																																																																																																																																																																																																																																																																																																																																																					<col>
																																																																																																																																																																																																																																																																																																																																																																																						<col width=\"200px\">
																																																																																																																																																																																																																																																																																																																																																																																							<tr>
																																																																																																																																																																																																																																																																																																																																																																																								<th colspan=\"2\">{{ 'groups_presets' | i18n:loc.ale:'farm_overflow' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																										<td>
																																																																																																																																																																																																																																																																																																																																																																																											<span class=\"ff-cell-fix\">{{ 'presets' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																											<td>
																																																																																																																																																																																																																																																																																																																																																																																												<div select=\"\" list=\"presets\" selected=\"settings[SETTINGS.PRESETS]\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																												<tr>
																																																																																																																																																																																																																																																																																																																																																																																													<td>
																																																																																																																																																																																																																																																																																																																																																																																														<span class=\"ff-cell-fix\">{{ 'group_ignored' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																														<td>
																																																																																																																																																																																																																																																																																																																																																																																															<div select=\"\" list=\"groupsWithDisabled\" selected=\"settings[SETTINGS.GROUP_IGNORE]\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																															<tr>
																																																																																																																																																																																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"ff-cell-fix\">{{ 'group_include' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																		<div select=\"\" list=\"groups\" selected=\"settings[SETTINGS.GROUP_INCLUDE]\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																																																																																																																			<td>
																																																																																																																																																																																																																																																																																																																																																																																																				<span class=\"ff-cell-fix\">{{ 'group_only' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																				<td>
																																																																																																																																																																																																																																																																																																																																																																																																					<div select=\"\" list=\"groups\" selected=\"settings[SETTINGS.GROUP_ONLY]\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																				</table>
																																																																																																																																																																																																																																																																																																																																																																																																				<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																																																																																																																																																																																																																																																																																																																																																																																																					<col>
																																																																																																																																																																																																																																																																																																																																																																																																						<col width=\"200px\">
																																																																																																																																																																																																																																																																																																																																																																																																							<col width=\"60px\">
																																																																																																																																																																																																																																																																																																																																																																																																								<tr>
																																																																																																																																																																																																																																																																																																																																																																																																									<th colspan=\"3\">{{ 'misc' | i18n:loc.ale:'farm_overflow' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																											<td>
																																																																																																																																																																																																																																																																																																																																																																																																												<span class=\"ff-cell-fix\">{{ 'attack_interval' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																												<td>
																																																																																																																																																																																																																																																																																																																																																																																																													<div range-slider=\"\" min=\"0\" max=\"30\" value=\"settings[SETTINGS.ATTACK_INTERVAL]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																													<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																														<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.ATTACK_INTERVAL]\">
																																																																																																																																																																																																																																																																																																																																																																																																															<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"ff-cell-fix\">{{ 'preserve_command_slots' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																																		<div range-slider=\"\" min=\"0\" max=\"50\" value=\"settings[SETTINGS.PRESERVE_COMMAND_SLOTS]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																		<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																			<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.PRESERVE_COMMAND_SLOTS]\">
																																																																																																																																																																																																																																																																																																																																																																																																																				<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																					<td colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																																																																																						<span class=\"ff-cell-fix\">{{ 'ignore_on_loss' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																						<td>
																																																																																																																																																																																																																																																																																																																																																																																																																							<div switch-slider=\"\" enabled=\"settings[SETTINGS.GROUP_IGNORE].value\" border=\"true\" value=\"settings[SETTINGS.IGNORE_ON_LOSS]\" vertical=\"false\" size=\"'56x28'\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																							<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																								<td colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																																																																																									<span class=\"ff-cell-fix\">{{ 'ignore_full_storage' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																									<td>
																																																																																																																																																																																																																																																																																																																																																																																																																										<div switch-slider=\"\" enabled=\"true\" border=\"true\" value=\"settings[SETTINGS.IGNORE_FULL_STORAGE]\" vertical=\"false\" size=\"'56x28'\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																									</table>
																																																																																																																																																																																																																																																																																																																																																																																																																									<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																																																																																																																																																																																																																																																																																																																																																																																																																										<col>
																																																																																																																																																																																																																																																																																																																																																																																																																											<col width=\"200px\">
																																																																																																																																																																																																																																																																																																																																																																																																																												<col width=\"60px\">
																																																																																																																																																																																																																																																																																																																																																																																																																													<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																														<th colspan=\"3\">{{ 'step_cycle_header' | i18n:loc.ale:'farm_overflow' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																<td colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"ff-cell-fix\">{{ 'target_single_attack' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																		<div switch-slider=\"\" enabled=\"true\" border=\"true\" value=\"settings[SETTINGS.TARGET_SINGLE_ATTACK]\" vertical=\"false\" size=\"'56x28'\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																			<td colspan=\"2\">
																																																																																																																																																																																																																																																																																																																																																																																																																																				<span class=\"ff-cell-fix\">{{ 'target_multiple_farmers' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																				<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																					<div switch-slider=\"\" enabled=\"true\" border=\"true\" value=\"settings[SETTINGS.TARGET_MULTIPLE_FARMERS]\" vertical=\"false\" size=\"'56x28'\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																					<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																						<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																							<span class=\"ff-cell-fix\">{{ 'farmer_cycle_interval' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																							<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																								<div range-slider=\"\" min=\"0\" max=\"120\" value=\"settings[SETTINGS.FARMER_CYCLE_INTERVAL]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																								<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																									<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.FARMER_CYCLE_INTERVAL]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																									</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																									<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																																																																																																																																																																																																																																																																																																																																																																																																																																										<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																											<col width=\"200px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																												<col width=\"60px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																													<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																														<th colspan=\"3\">{{ 'target_filters' | i18n:loc.ale:'farm_overflow' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"ff-cell-fix\">{{ 'min_distance' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																		<div range-slider=\"\" min=\"0\" max=\"50\" value=\"settings[SETTINGS.MIN_DISTANCE]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																		<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																			<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.MIN_DISTANCE]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																				<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																					<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																						<span class=\"ff-cell-fix\">{{ 'max_distance' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																						<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																							<div range-slider=\"\" min=\"0\" max=\"50\" value=\"settings[SETTINGS.MAX_DISTANCE]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																							<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																								<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.MAX_DISTANCE]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																									<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																										<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																											<span class=\"ff-cell-fix\">{{ 'min_points' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																											<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																												<div range-slider=\"\" min=\"0\" max=\"13000\" value=\"settings[SETTINGS.MIN_POINTS]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																												<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																													<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.MIN_POINTS]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																														<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																															<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																<span class=\"ff-cell-fix\">{{ 'max_points' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<div range-slider=\"\" min=\"0\" max=\"13000\" value=\"settings[SETTINGS.MAX_POINTS]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.MAX_POINTS]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<span class=\"ff-cell-fix\">{{ 'max_travel_time' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<div range-slider=\"\" min=\"0\" max=\"180\" value=\"settings[SETTINGS.MAX_TRAVEL_TIME]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.MAX_TRAVEL_TIME]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																							</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<table class=\"tbl-border-light tbl-content tbl-medium-height\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<col width=\"200px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<col width=\"60px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<th colspan=\"3\">{{ 'others' | i18n:loc.ale:'common' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<span class=\"ff-cell-fix\">{{ 'logs_limit' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<div range-slider=\"\" min=\"0\" max=\"1000\" value=\"settings[SETTINGS.LOGS_LIMIT]\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<td class=\"cell-bottom\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<input class=\"fit textfield-border text-center\" ng-model=\"settings[SETTINGS.LOGS_LIMIT]\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<div class=\"villages rich-text\" ng-show=\"selectedTab === TAB_TYPES.VILLAGES\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<h5 class=\"twx-section\">{{ 'farmer_villages' | i18n:loc.ale:'farm_overflow' }}</h5>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<p ng-if=\"!farmers.length\" class=\"text-center\">{{ 'no_farmer_villages' | i18n:loc.ale:'farm_overflow' }}<table class=\"tbl-border-light tbl-striped\" ng-show=\"farmers.length\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<col width=\"40%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<col width=\"20%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<th>{{ 'villages' | i18n:loc.ale:'common' }}<th>{{ 'last_status' | i18n:loc.ale:'farm_overflow' }}<th>{{ 'target' | i18n:loc.ale:'common':2 }}<tr ng-repeat=\"farmer in farmers\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<span ng-class=\"{true:'icon-20x20-queue-indicator-long', false:'icon-20x20-queue-indicator-short'}[farmer.isRunning()]\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<a class=\"link\" ng-click=\"openVillageInfo(farmer.getVillage().getId())\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<span class=\"icon-20x20-village\"/> {{ farmer.getVillage().getName() }} ({{ farmer.getVillage().getX() }}|{{ farmer.getVillage().getY() }})</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<td>{{ 'status_' + farmer.getStatus() | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"farmer.getTargets()\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<span ng-if=\"farmer.isRunning()\">{{ farmer.getIndex() }} / </span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<span>{{ farmer.getTargets().length }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<td ng-if=\"!farmer.getTargets()\">{{ 'not_loaded' | i18n:loc.ale:'farm_overflow' }}</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<h5 class=\"twx-section\">{{ 'ignored_targets' | i18n:loc.ale:'farm_overflow' }}</h5>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<p ng-if=\"!exceptionVillages.ignored.length\" class=\"text-center\">{{ 'no_ignored_targets' | i18n:loc.ale:'farm_overflow' }}<table class=\"ignored-villages tbl-border-light tbl-striped\" ng-show=\"exceptionVillages.ignored.length\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<col width=\"15%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<col width=\"15%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<col width=\"30px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<th>{{ 'villages' | i18n:loc.ale:'common' }}<th>{{ 'date' | i18n:loc.ale:'farm_overflow' }}<th>{{ 'reports' | i18n:loc.ale:'farm_overflow' }}<th>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<tr ng-repeat=\"villageId in exceptionVillages.ignored track by $index\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<a class=\"link\" ng-click=\"openVillageInfo(villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<span class=\"icon-20x20-village\"/> {{ villagesLabel[villageId] }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<td>{{ exceptionLogs[villageId].time | readableDateFilter:loc.ale:GAME_TIMEZONE:GAME_TIME_OFFSET }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<span ng-if=\"exceptionLogs[villageId].report\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<a class=\"link\" ng-click=\"showReport(exceptionLogs[villageId].report.id)\" tooltip=\"\" tooltip-content=\"{{ exceptionLogs[villageId].report.title }}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<span class=\"icon-20x20-report\"/> {{ 'open_report' | i18n:loc.ale:'farm_overflow' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<span ng-class=\"{2:'icon-20x20-queue-indicator-medium', 3:'icon-20x20-queue-indicator-short'}[exceptionLogs[villageId].report.result]\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<span ng-class=\"{'full': 'icon-26x26-capacity', 'partial':'icon-26x26-capacity-low', 'none':'hidden'}[exceptionLogs[villageId].report.haul]\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<span ng-if=\"!exceptionLogs[villageId].report\">{{ 'no_report' | i18n:loc.ale:'farm_overflow' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<a href=\"#\" class=\"size-20x20 btn-red icon-20x20-close\" ng-click=\"removeIgnored(villageId)\" tooltip=\"\" tooltip-content=\"\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<h5 class=\"twx-section\">{{ 'included_targets' | i18n:loc.ale:'farm_overflow' }}</h5>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<p ng-if=\"!exceptionVillages.included.length\" class=\"text-center\">{{ 'no_included_targets' | i18n:loc.ale:'farm_overflow' }}<table class=\"tbl-border-light tbl-striped\" ng-show=\"exceptionVillages.included.length\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<col width=\"15%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<col width=\"30px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<th>{{ 'villages' | i18n:loc.ale:'common' }}<th>{{ 'date' | i18n:loc.ale:'farm_overflow' }}<th>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<tr ng-repeat=\"villageId in exceptionVillages.included track by $index\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<a class=\"link\" ng-click=\"openVillageInfo(villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<span class=\"icon-20x20-village\"/> {{ villagesLabel[villageId] }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<td>{{ exceptionLogs[villageId].time | readableDateFilter:loc.ale:GAME_TIMEZONE:GAME_TIME_OFFSET }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<a href=\"#\" class=\"size-20x20 btn-red icon-20x20-close\" ng-click=\"removeIncluded(villageId)\" tooltip=\"\" tooltip-content=\"\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<div class=\"logs rich-text\" ng-show=\"selectedTab === TAB_TYPES.LOGS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<div class=\"page-wrap\" pagination=\"pagination\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<p class=\"text-center\" ng-show=\"!visibleLogs.length\">{{ 'no_logs' | i18n:loc.ale:'farm_overflow' }}<table class=\"log-list tbl-border-light tbl-striped\" ng-show=\"visibleLogs.length\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<col width=\"100px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<col width=\"30px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<tr ng-repeat=\"log in visibleLogs track by $index\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<td>{{ log.time | readableDateFilter:loc.ale:GAME_TIMEZONE:GAME_TIME_OFFSET }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"icon-bg-black\" ng-class=\"{'icon-26x26-dot-green': log.type === LOG_TYPES.FARM_START, 'icon-26x26-dot-red': log.type === LOG_TYPES.FARM_STOP, 'icon-26x26-check-negative': log.type === LOG_TYPES.IGNORED_VILLAGE || log.type === LOG_TYPES.INCLUDED_VILLAGE_REMOVED, 'icon-26x26-check-positive': log.type === LOG_TYPES.INCLUDED_VILLAGE || log.type === LOG_TYPES.IGNORED_VILLAGE_REMOVED}\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<td ng-if=\"log.type === LOG_TYPES.IGNORED_VILLAGE\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<a class=\"link\" ng-click=\"openVillageInfo(log.villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<span class=\"icon-20x20-village\"/> {{ villagesLabel[log.villageId] }}</a> {{ 'ignored_village' | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"log.type === LOG_TYPES.IGNORED_VILLAGE_REMOVED\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<a class=\"link\" ng-click=\"openVillageInfo(log.villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<span class=\"icon-20x20-village\"/> {{ villagesLabel[log.villageId] }}</a> {{ 'ignored_village_removed' | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"log.type === LOG_TYPES.INCLUDED_VILLAGE\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<a class=\"link\" ng-click=\"openVillageInfo(log.villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<span class=\"icon-20x20-village\"/> {{ villagesLabel[log.villageId] }}</a> {{ 'included_village' | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"log.type === LOG_TYPES.INCLUDED_VILLAGE_REMOVED\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<a class=\"link\" ng-click=\"openVillageInfo(log.villageId)\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<span class=\"icon-20x20-village\"/> {{ villagesLabel[log.villageId] }}</a> {{ 'included_village_removed' | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"log.type === LOG_TYPES.FARM_START\">{{ 'farm_started' | i18n:loc.ale:'farm_overflow' }}<td ng-if=\"log.type === LOG_TYPES.FARM_STOP\">{{ 'farm_stopped' | i18n:loc.ale:'farm_overflow' }}</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<div class=\"page-wrap\" pagination=\"pagination\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<footer class=\"win-foot\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<li ng-show=\"selectedTab === TAB_TYPES.SETTINGS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"saveSettings()\">{{ 'save' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<li ng-show=\"selectedTab === TAB_TYPES.LOGS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<a href=\"#\" class=\"btn-border btn-orange\" ng-click=\"clearLogs()\">{{ 'clear_logs' | i18n:loc.ale:'farm_overflow' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<li>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<a href=\"#\" ng-class=\"{false:'btn-green', true:'btn-red'}[running]\" class=\"btn-border\" ng-click=\"switchFarm()\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<span ng-show=\"running\">{{ 'pause' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<span ng-show=\"!running\">{{ 'start' | i18n:loc.ale:'common' }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						</ul>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					</footer>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				</div>`)
        interfaceOverflow.addStyle('#two-farmoverflow .settings table{margin-bottom:15px}#two-farmoverflow .settings input.textfield-border{padding-top:6px;width:219px;height:34px}#two-farmoverflow .settings input.textfield-border.fit{width:100%}#two-farmoverflow .settings span.select-wrapper{width:219px}#two-farmoverflow .settings .range-container{width:350px}#two-farmoverflow .villages td{padding:2px 5px;white-space:nowrap}#two-farmoverflow .villages .hidden{display:none}#two-farmoverflow .logs .status tr{height:25px}#two-farmoverflow .logs .status td{padding:0 6px}#two-farmoverflow .logs .log-list{margin-bottom:10px}#two-farmoverflow .logs .log-list td{white-space:nowrap;text-align:center;padding:0 5px}#two-farmoverflow .logs .log-list td .village-link{max-width:200px;white-space:nowrap;text-overflow:ellipsis}#two-farmoverflow a.link{font-weight:bold;color:#3f2615}#two-farmoverflow a.link:hover{text-shadow:0 1px 0 #000;color:#fff}#two-farmoverflow .icon-20x20-village:before{margin-top:-11px}')
    }

    const buildWindow = function () {
        $scope = $rootScope.$new()
        $scope.SETTINGS = SETTINGS
        $scope.TAB_TYPES = TAB_TYPES
        $scope.LOG_TYPES = LOG_TYPES
        $scope.running = farmOverflow.isRunning()
        $scope.selectedTab = TAB_TYPES.SETTINGS
        $scope.farmers = farmOverflow.getAll()
        $scope.villagesLabel = villagesLabel
        $scope.villagesInfo = villagesInfo
        $scope.exceptionVillages = farmOverflow.getExceptionVillages()
        $scope.exceptionLogs = farmOverflow.getExceptionLogs()
        $scope.logs = farmOverflow.getLogs()
        $scope.visibleLogs = []

        $scope.pagination = {
            count: $scope.logs.length,
            offset: 0,
            loader: updateVisibleLogs,
            limit: storageService.getPaginationLimit()
        }

        settings.injectScope($scope)
        eventHandlers.updatePresets()
        eventHandlers.updateGroups()
        updateVisibleLogs()
        loadExceptionsInfo()

        // scope functions
        $scope.switchFarm = switchFarm
        $scope.selectTab = selectTab
        $scope.saveSettings = saveSettings
        $scope.clearLogs = farmOverflow.clearLogs
        $scope.jumpToVillage = mapService.jumpToVillage
        $scope.openVillageInfo = windowDisplayService.openVillageInfo
        $scope.showReport = reportService.showReport
        $scope.removeIgnored = removeIgnored
        $scope.removeIncluded = removeIncluded

        let eventScope = new EventScope('twoverflow_farm_overflow_window')
        eventScope.register(eventTypeProvider.ARMY_PRESET_UPDATE, eventHandlers.updatePresets, true)
        eventScope.register(eventTypeProvider.ARMY_PRESET_DELETED, eventHandlers.updatePresets, true)
        eventScope.register(eventTypeProvider.GROUPS_UPDATED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.GROUPS_CREATED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.GROUPS_DESTROYED, eventHandlers.updateGroups, true)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_START, eventHandlers.start)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_STOP, eventHandlers.stop)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_LOGS_UPDATED, eventHandlers.updateLogs)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_FARMER_VILLAGES_UPDATED, eventHandlers.updateFarmerVillages)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_EXCEPTION_VILLAGES_UPDATED, eventHandlers.updateExceptionVillages)
        eventScope.register(eventTypeProvider.FARM_OVERFLOW_EXCEPTION_LOGS_UPDATED, eventHandlers.updateExceptionLogs)

        windowManagerService.getScreenWithInjectedScope('!twoverflow_farm_overflow_window', $scope)
    }

    return init
})

define('two/farmOverflow/settings', [], function () {
    return {
        PRESETS: 'presets',
        GROUP_IGNORE: 'group_ignore',
        GROUP_INCLUDE: 'group_include',
        GROUP_ONLY: 'group_only',
        MAX_DISTANCE: 'max_distance',
        MIN_DISTANCE: 'min_distance',
        IGNORE_FULL_STORAGE: 'ignore_full_storage',        
        ATTACK_INTERVAL: 'attack_interval',
        MAX_TRAVEL_TIME: 'max_travel_time',
        TARGET_SINGLE_ATTACK: 'target_single_attack',
        TARGET_MULTIPLE_FARMERS: 'target_multiple_farmers',
        PRESERVE_COMMAND_SLOTS: 'preserve_command_slots',
        FARMER_CYCLE_INTERVAL: 'farmer_cycle_interval',
        MIN_POINTS: 'min_points',
        MAX_POINTS: 'max_points',
        LOGS_LIMIT: 'logs_limit',
        IGNORE_ON_LOSS: 'ignore_on_loss'
    }
})

define('two/farmOverflow/settings/updates', function () {
    return {
        PRESET: 'preset',
        GROUPS: 'groups',
        TARGETS: 'targets',
        VILLAGES: 'villages',
        WAITING_VILLAGES: 'waiting_villages',
        FULL_STORAGE: 'full_storage',
        LOGS: 'logs',
        INTERVAL_TIMERS: 'interval_timers'
    }
})

define('two/farmOverflow/settings/map', [
    'two/farmOverflow/settings',
    'two/farmOverflow/settings/updates'
], function (
    SETTINGS,
    UPDATES
) {
    return {
        [SETTINGS.PRESETS]: {
            default: [],
            updates: [
                UPDATES.PRESET,
                UPDATES.INTERVAL_TIMERS
            ],
            disabledOption: true,
            inputType: 'select',
            multiSelect: true,
            type: 'presets'
        },
        [SETTINGS.GROUP_IGNORE]: {
            default: false,
            updates: [
                UPDATES.GROUPS,
                UPDATES.INTERVAL_TIMERS
            ],
            disabledOption: true,
            inputType: 'select',
            type: 'groups'
        },
        [SETTINGS.GROUP_INCLUDE]: {
            default: [],
            updates: [
                UPDATES.GROUPS,
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ],
            disabledOption: true,
            inputType: 'select',
            multiSelect: true,
            type: 'groups'
        },
        [SETTINGS.GROUP_ONLY]: {
            default: [],
            updates: [
                UPDATES.GROUPS,
                UPDATES.VILLAGES,
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ],
            disabledOption: true,
            inputType: 'select',
            multiSelect: true,
            type: 'groups'
        },
        [SETTINGS.ATTACK_INTERVAL]: {
            default: 1,
            updates: [UPDATES.INTERVAL_TIMERS]
        },
        [SETTINGS.FARMER_CYCLE_INTERVAL]: {
            default: 5,
            updates: [UPDATES.INTERVAL_TIMERS]
        },
        [SETTINGS.TARGET_SINGLE_ATTACK]: {
            default: true,
            updates: [],
            inputType: 'checkbox'
        },
        [SETTINGS.TARGET_MULTIPLE_FARMERS]: {
            default: true,
            updates: [UPDATES.INTERVAL_TIMERS],
            inputType: 'checkbox'
        },
        [SETTINGS.PRESERVE_COMMAND_SLOTS]: {
            default: 5,
            updates: []
        },
        [SETTINGS.IGNORE_ON_LOSS]: {
            default: true,
            updates: [],
            inputType: 'checkbox'
        },
        [SETTINGS.IGNORE_FULL_STORAGE]: {
            default: true,
            updates: [UPDATES.INTERVAL_TIMERS],
            inputType: 'checkbox'
        },
        [SETTINGS.MIN_DISTANCE]: {
            default: 0,
            updates: [
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ]
        },
        [SETTINGS.MAX_DISTANCE]: {
            default: 15,
            updates: [
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ]
        },
        [SETTINGS.MIN_POINTS]: {
            default: 0,
            updates: [
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ]
        },
        [SETTINGS.MAX_POINTS]: {
            default: 12500,
            updates: [
                UPDATES.TARGETS,
                UPDATES.INTERVAL_TIMERS
            ]
        },
        [SETTINGS.MAX_TRAVEL_TIME]: {
            default: 90,
            updates: [UPDATES.INTERVAL_TIMERS]
        },
        [SETTINGS.LOGS_LIMIT]: {
            default: 500,
            updates: [UPDATES.LOGS]
        }
    }
})

define('two/farmOverflow/types/errors', [], function () {
    return {
        NO_PRESETS: 'no_presets',
        USER_STOP: 'user_stop'
    }
})

define('two/farmOverflow/types/status', [], function () {
    return {
        TIME_LIMIT: 'time_limit',
        COMMAND_LIMIT: 'command_limit',
        FULL_STORAGE: 'full_storage',
        NO_UNITS: 'no_units',
        NO_SELECTED_VILLAGE: 'no_selected_village',
        ABANDONED_CONQUERED: 'abandoned_conquered',
        PROTECTED_VILLAGE: 'protected_village',
        BUSY_TARGET: 'busy_target',
        NO_TARGETS: 'no_targets',
        TARGET_CYCLE_END: 'target_cycle_end',
        FARMER_CYCLE_END: 'farmer_cycle_end',
        COMMAND_ERROR: 'command_error',
        NOT_ALLOWED_POINTS: 'not_allowed_points',
        UNKNOWN: 'unknown',
        ATTACKING: 'attacking',
        WAITING_CYCLE: 'waiting_cycle'
    }
})

define('two/farmOverflow/types/logs', [], function () {
    return {
        FARM_START: 'farm_start',
        FARM_STOP: 'farm_stop',
        IGNORED_VILLAGE: 'ignored_village',
        INCLUDED_VILLAGE: 'included_village',
        IGNORED_VILLAGE_REMOVED: 'ignored_village_removed',
        INCLUDED_VILLAGE_REMOVED: 'included_village_removed'
    }
})

require([
    'two/language',
    'two/ready',
    'two/farmOverflow',
    'two/farmOverflow/ui',
    'two/farmOverflow/events'
], function (
    twoLanguage,
    ready,
    farmOverflow,
    farmOverflowInterface
) {
    if (farmOverflow.isInitialized()) {
        return false
    }

    ready(function () {
        twoLanguage.add('farm_overflow', {"de_de":{"farm_overflow":{"open_report":"Bericht 철ffnen","no_report":"Kein Bericht","reports":"Berichte","date":"Datum","status_time_limit":"Ziel ist zu weit weg","status_command_limit":"Befehlslimit","status_full_storage":"Speicher voll","status_no_units":"Keine verf체gbaren Einheiten","status_abandoned_conquered":"Keine vef체gbaren Einheiten","status_protected_village":"Ziel ist gesch체tzt","status_busy_target":"Ziel wird angegriffen","status_no_targets":"Keine verf체gbaren Ziele","status_target_cycle_end":"Zielzyklus beendet","status_not_allowed_points":"Zielpunkt nicht erlaubt","status_unknown":"Unbekannter Status","status_attacking":"Angreifen","status_waiting_cycle":"Wartezyklus","not_loaded":"Nicht geladen.","ignored_targets":"Ignorierte Ziele","no_ignored_targets":"Nichts ignoriert","included_targets":"Hinzugef체gte Ziele","no_included_targets":"Nichts hinzugef체gt","farmer_villages":"Farmd철rfer","no_farmer_villages":"Keine Farmd철rfer","last_status":"Letzter Status","attacking":"Im Angriff.","paused":"Angehalten.","command_limit":"Das Befehlslimit ist erreicht, bitte warte auf die R체ckkehr einiger Trupps.","last_attack":"Letzter Angriff","village_switch":"Wechsle zu Dorf","no_preset":"Keine Vorlage verf체gbar.","no_selected_village":"Keine D철rfer verf체gbar.","no_units":"Keine Einheiten im Dorf, wartet auf die R체ckkehr von Trupps.","no_units_no_commands":"Kein Dorf hat Einheiten oder zur체ckkehrende Befehle.","no_villages":"Keine D철rfer verf체gbar, warte auf R체ckkehr von Befehlen.","preset_first":"W채hle zuerst eine Vorlage!","selected_village":"Ausgew채hltes Dorf","loading_targets":"Ziele werden geladen...","checking_targets":"Ziele werden gepr체ft...","restarting_commands":"Befehle werden neu gestartet...","ignored_village":"zur Ignorierliste hinzugef체gt","included_village":"zur Eingeschlossenen-Liste hinzugef체gt","ignored_village_removed":"aus der Ignorier-Liste entfernt","included_village_removed":"aus der Eingeschlossenen-Liste entfernt","priority_target":"zur Priorit채tenliste hinzugef체gt.","analyse_targets":"Ziele werden analysiert.","step_cycle_restart":"Befehlszyklus wird neu gestartet..","step_cycle_end":"Dorfliste abgearbeitet, warte auf den n채chsten Durchlauf.","step_cycle_end_no_villages":"Keine D철rfer verf체gbar um den Durchgang zu starten.","step_cycle_next":"Dorfliste abgearbeitet, n채chster Durchgang: %d.","step_cycle_next_no_villages":"Kein Dorf verf체gbar um den Durchgang zu starten, n채chster Durchgang: %d.","full_storage":"Der Speicher des Dorfes ist voll.","farm_stopped":"FarmOverflow angehalten.","farm_started":"FarmOverflow gestartet.","groups_presets":"Gruppen & Vorlagen","presets":"Vorlagen zum Angriff","group_ignored":"Ignoriere Dorfgruppe","group_include":"Dorfgruppe umfassen","group_only":"Greife nur aus D철rfern dieser Gruppen an","attack_interval":"Angriffsintervall","preserve_command_slots":"Befehlsslot aufbewahren","target_single_attack":"Erlaube nur einen Dorf-Angriff pro Ziel","target_multiple_farmers":"Erlaube Angriffe aus mehreren D철rfern pro Ziel","farmer_cycle_interval":"Intervall zwischen Farmzyklen (in Min.)","ignore_on_loss":"Ignoriere D철rfer, die Truppenverluste verursachen","ignore_full_storage":"D철rfer mit vollen Speichern 체berspringen","step_cycle_header":"Farmzyklus","step_cycle":"Farmzyklus aktivieren","step_cycle_notifs":"Benachrichtigungen","target_filters":"Zielfilter","min_distance":"Minimale Distanz des Zieldorfes","max_distance":"Maximale Distanz des Zieldorfes","min_points":"Minimale Punktzahl des Zielsdorfes","max_points":"Minimale Punktzahl des Zieldorfes","max_travel_time":"Maximale Laufzeit zum Zieldorf (Minuten)","logs_limit":"Max. Anzahl an Protokoll-Eintr채gen","event_attack":"Angriffe protokollieren","event_village_change":"Dorfwechsel protokollieren","event_priority_add":"Priorisierte Ziele protokollieren","event_ignored_village":"Ignorierte D철rfer protokollieren","settings_saved":"Einstellungen gespeichert!","misc":"Weitere Einstellungen","attack":"angreifen","no_logs":"Keine Angriffe protokolliert","clear_logs":"Protokoll l철schen","reseted_logs":"Protokoll wurde zur체ckgesetzt.","date_added":"Datum hinzugef체gt"}},"en_us":{"farm_overflow":{"open_report":"Open report","no_report":"No report","reports":"Reports","date":"Date","status_time_limit":"Target is too far","status_command_limit":"Command limit","status_full_storage":"Storage is full","status_no_units":"No units available","status_abandoned_conquered":"Abandoned conquered","status_protected_village":"Target is protected","status_busy_target":"Target is under attack","status_no_targets":"No targets available","status_target_cycle_end":"Target cycle ended","status_not_allowed_points":"Target points not allowed","status_unknown":"Unknown status","status_attacking":"Attacking","status_waiting_cycle":"Waiting cycle","not_loaded":"Not loaded.","ignored_targets":"Ignored Targets","no_ignored_targets":"Nothing ignored","included_targets":"Included Targets","no_included_targets":"Nothing included","farmer_villages":"Farmer Villages","no_farmer_villages":"No farmer villages","last_status":"Last Status","attacking":"Attacking.","paused":"Paused.","command_limit":"Limit of 50 attacks reached, waiting return.","last_attack":"Last attack","village_switch":"Changing to village","no_preset":"No presets avaliable.","no_selected_village":"No villages avaliable.","no_units":"No units avaliable in village, waiting attacks return.","no_units_no_commands":"No villages has units or commands returning.","no_villages":"No villages avaliable, waiting attacks return.","preset_first":"Set a preset first!","selected_village":"Village selected","loading_targets":"Loading targets...","checking_targets":"Checking targets...","restarting_commands":"Restarting commands...","ignored_village":"added to the ignored list","included_village":"added to the included list","ignored_village_removed":"removed from the ignored list","included_village_removed":"removed from the included list","priority_target":"added to priorities.","analyse_targets":"Analysing targets.","step_cycle_restart":"Restarting the cycle of commands..","step_cycle_end":"The list of villages ended, waiting for the next run.","step_cycle_end_no_villages":"No villages available to start the cycle.","step_cycle_next":"The list of villages is over, next cycle: %d.","step_cycle_next_no_villages":"No village available to start the cycle, next cycle: %d.","full_storage":"The storage of the village is full.","farm_stopped":"FarmOverflow stopped.","farm_started":"FarmOverflow started.","groups_presets":"Groups & presets","presets":"Attack with the presets","group_ignored":"Ignore villages from group","group_include":"Include villages from groups","group_only":"Only attack with villages from groups","attack_interval":"Interval between attacks","preserve_command_slots":"Preserve command slots","target_single_attack":"Allow targets to receive only one attack per village","target_multiple_farmers":"Allow targets to receive attacks from multiple villages","farmer_cycle_interval":"Interval between farmer cycles (minutes)","ignore_on_loss":"Ignore target that cause loss","ignore_full_storage":"Do not farm with villages with full storage","step_cycle_header":"Step Cycle Settings","step_cycle":"Enable Step Cycle","step_cycle_notifs":"Cycle notifications","target_filters":"Target Filters","min_distance":"Targets minimum distance","max_distance":"Targets maximum distance","min_points":"Targets minimum points","max_points":"Targets maximum points","max_travel_time":"Maximum travel time (minutes)","logs_limit":"Maximum amount of log entries","event_attack":"Show task logs of attacks","event_village_change":"Show task logs of village's changes","event_priority_add":"Show task logs of priority targets","event_ignored_village":"Show task logs of ignored villages","settings_saved":"Settings saved!","misc":"Miscellaneous","attack":"attack","no_logs":"No logs registered","clear_logs":"Clear logs","reseted_logs":"Registered logs reseted.","date_added":"Date added"}},"pl_pl":{"farm_overflow":{"open_report":"Open report","no_report":"No report","reports":"Reports","date":"Date","status_time_limit":"Target is too far","status_command_limit":"Command limit","status_full_storage":"Storage is full","status_no_units":"No units available","status_abandoned_conquered":"Abandoned conquered","status_protected_village":"Target is protected","status_busy_target":"Target is under attack","status_no_targets":"No targets available","status_target_cycle_end":"Target cycle ended","status_not_allowed_points":"Target points not allowed","status_unknown":"Unknown status","status_attacking":"Attacking","status_waiting_cycle":"Waiting cycle","not_loaded":"Not loaded.","ignored_targets":"Ignored Targets","no_ignored_targets":"Nothing ignored","included_targets":"Included Targets","no_included_targets":"Nothing included","farmer_villages":"Farmer Villages","no_farmer_villages":"No farmer villages","last_status":"Last Status","attacking":"Atakuje.","paused":"Zatrzymany.","command_limit":"Limit 50 atak처w osi훳gni휌ty, oczekiwanie na powr처t wojsk.","last_attack":"Ostatni atak","village_switch":"Przej힄cie do wioski","no_preset":"Brak dost휌pnych szablon처w.","no_selected_village":"Brak dost휌pnych wiosek.","no_units":"Brak dost휌pnych jednostek w wiosce, oczekiwanie na powr처t wojsk.","no_units_no_commands":"Brak jednostek w wioskach lub powracaj훳cych wojsk.","no_villages":"Brak dost휌pnych wiosek, oczekiwanie na powr처t wojsk.","preset_first":"Wybierz najpierw szablon!","selected_village":"Wybrana wioska","loading_targets":"흟adowanie cel처w...","checking_targets":"Sprawdzanie cel처w...","restarting_commands":"Restartowanie polece흦...","ignored_village":"dodany do listy pomini휌tych","included_village":"added to the included list","ignored_village_removed":"removed from the ignored list","included_village_removed":"removed from the included list","priority_target":"dodany do priorytetowych.","analyse_targets":"Analizowanie cel처w.","step_cycle_restart":"Restartowanie cyklu polece흦...","step_cycle_end":"Lista wiosek zako흦czona, oczekiwanie na nast휌pny cykl.","step_cycle_end_no_villages":"Brak wiosek do rozpocz휌cia cyklu.","step_cycle_next":"Lista wiosek si휌 sko흦czy흢a, nast휌pny cykl: %d.","step_cycle_next_no_villages":"Brak wioski do rozpocz휌cia cyklu, nast휌pny cykl: %d.","full_storage":"Magazyn w wiosce jest pe흢ny","farm_stopped":"FarmOverflow stopped.","farm_started":"Farmer uruchomiony","groups_presets":"Grupy i szablony","presets":"Szablony","group_ignored":"Pomijaj wioski z grupy","group_include":"Dodaj wioski z grupy","group_only":"Only attack with villages from groups","attack_interval":"Interval between attacks","preserve_command_slots":"Preserve command slots","target_single_attack":"Allow targets to receive only one attack per village","target_multiple_farmers":"Allow targets to receive attacks from multiple villages","farmer_cycle_interval":"Interval between farmer cycles (minutes)","ignore_on_loss":"Pomijaj cele je힄li straty","ignore_full_storage":"Pomijaj wioski je힄li magazyn pe흢ny","step_cycle_header":"Cykl Farmienia","step_cycle":"W흢훳cz Cykl farmienia","step_cycle_notifs":"Powiadomienia","target_filters":"Filtry cel처w","min_distance":"Minimalna odleg흢o힄훶","max_distance":"Maksymalna odleg흢o힄훶","min_points":"Minimalna liczba punkt처w","max_points":"Maksymalna liczba punkt처w","max_travel_time":"Maksymalny czas podr처탉y (minuty)","logs_limit":"Maximum amount of log entries","event_attack":"Logi atak처w","event_village_change":"Logi zmiany wiosek","event_priority_add":"Logi cel처w priorytetowych","event_ignored_village":"Logi pomini휌tych wiosek","settings_saved":"Ustawienia zapisane!","misc":"R처탉ne","attack":"atakuje","no_logs":"Brak zarejestrowanych log처w","clear_logs":"Wyczy힄훶 logi","reseted_logs":"Zarejestrowane logi zosta흢y wyczyszczone.","date_added":"Date added"}},"pt_br":{"farm_overflow":{"open_report":"Abrir relat처rio","no_report":"Sem relat처rio","reports":"Relat처rios","date":"Data","status_time_limit":"Alvo muito longe","status_command_limit":"Limite de comandos","status_full_storage":"Armaz챕m lotado","status_no_units":"Sem tropas dispon챠veis","status_abandoned_conquered":"Abandonada conquistada","status_protected_village":"Alvo est찼 protegido","status_busy_target":"Alvo j찼 sob ataque","status_no_targets":"Nenhum alvo dispon챠vel","status_target_cycle_end":"Cicle de alvos acabou","status_not_allowed_points":"Pontos do alvo n찾o permitido","status_unknown":"Status desconhecido","status_attacking":"Atacando","status_waiting_cycle":"Aguardando ciclo","not_loaded":"N찾o carregado","ignored_targets":"Alvos Ignorados","no_ignored_targets":"Nada ignorado","included_targets":"Alvos inclu챠dos","no_included_targets":"Nada inclu챠do","farmer_villages":"Aldeias Farm","no_farmer_villages":"Nenhuma aldeia farm dispon챠vel","last_status":"횣ltimo Status","attacking":"Atacando.","paused":"Pausado.","command_limit":"Limite de 50 ataques atingido, aguardando retorno.","last_attack":"횣ltimo ataque","village_switch":"Alternando para a aldeia","no_preset":"Nenhuma predefini챌찾o dispon챠vel.","no_selected_village":"Nenhuma aldeia dispon챠vel.","no_units":"Sem unidades na aldeia, aguardando ataques retornarem.","no_units_no_commands":"Nenhuma aldeia tem tropas nem ataques retornando.","no_villages":"Nenhuma aldeia dispon챠vel, aguardando ataques retornarem.","preset_first":"Configure uma predefini챌찾o primeiro!","selected_village":"Aldeia selecionada","loading_targets":"Carregando alvos...","checking_targets":"Checando alvos...","restarting_commands":"Reiniciando comandos...","ignored_village":"adicionado a lista de ignorados","included_village":"adicionado 횪 lista de aldeias inclu챠das","ignored_village_removed":"removida da lista de aldeias ignoradas","included_village_removed":"removida da lista de aldeias inclu챠das","priority_target":"adicionado as prioridades.","analyse_targets":"Analisando alvos.","step_cycle_restart":"Reiniciando o ciclo de comandos..","step_cycle_end":"A lista de aldeias acabou, esperando pr처xima execu챌찾o.","step_cycle_end_no_villages":"Nenhuma aldeia dispon챠vel para iniciar o ciclo.","step_cycle_next":"A lista de aldeias acabou, pr처ximo ciclo: %d.","step_cycle_next_no_villages":"Nenhuma aldeia dispon챠vel para iniciar o ciclo, pr처ximo ciclo: %d.","full_storage":"O armaz챕m da aldeia est찼 cheio.","farm_stopped":"FarmOverflow parado.","farm_started":"FarmOverflow iniciado.","groups_presets":"Grupos & predefini챌천es","presets":"Atacar com as predefini챌천es","group_ignored":"Ignorar aldeias do grupo","group_include":"Incluir aldeias dos grupos","group_only":"Atacar apenas com aldeias dos grupos","attack_interval":"Intervalo entre ataques","preserve_command_slots":"N첬mero de comandos a se preservar","target_single_attack":"Permitir alvos receber apenas um ataque por aldeia","target_multiple_farmers":"Permitir alvos a receberem ataques de m첬ltiplas aldeias","farmer_cycle_interval":"Intervalo entre ciclos de farm (minutos)","ignore_on_loss":"Ignorar alvos que causarem perdas","ignore_full_storage":"Ignorar aldeias com armaz챕m lotado","step_cycle_header":"Configura챌천es de Ciclos","step_cycle":"Ativar Ciclo","step_cycle_notifs":"Notifica챌천es de ciclos","target_filters":"Filtro de Alvos","min_distance":"Dist창ncia m챠nima","max_distance":"Dist창ncia m찼xima","min_points":"Pontua챌찾o m챠nima","max_points":"Pontua챌찾o m찼xima","max_travel_time":"Tempo m찼ximo de viagem (minutos)","logs_limit":"Quantidade m찼xima de registros","event_attack":"Registrar ataques","event_village_change":"Registrar troca de aldeias","event_priority_add":"Registrar alvos prioritarios","event_ignored_village":"Registrar alvos ignorados","settings_saved":"Configura챌천es salvas!","misc":"Diversos","attack":"ataca","no_logs":"Nenhum evento registrado","clear_logs":"Limpar eventos","reseted_logs":"Registro de eventos resetado.","date_added":"Adicionado na data"}},"ru_ru":{"farm_overflow":{"open_report":"��克���� 棘��筠�","no_report":"�筠� 棘��筠�逵","reports":"������","date":"�逵�逵","status_time_limit":"揆筠剋� �剋龜�克棘劇 畇逵剋筠克棘","status_command_limit":"��筠畇筠剋 克棘剋龜�筠��勻逵 克棘劇逵戟畇","status_full_storage":"鬼克剋逵畇 極筠�筠極棘剋戟筠戟","status_no_units":"�筠� 畇棘���極戟�� 勻棘橘�克","status_abandoned_conquered":"�棘閨筠畇逵 畇筠�筠勻戟龜 勻逵�勻逵�棘勻","status_protected_village":"揆筠剋� 極棘畇 鈞逵�龜�棘橘","status_busy_target":"揆筠剋� 極棘畇 逵�逵克棘橘","status_no_targets":"�筠� 畇棘���極戟�� �筠剋筠橘","status_target_cycle_end":"�筠�筠�筠戟� �筠剋筠橘 鈞逵克棘戟�筠戟","status_not_allowed_points":"�逵極�筠�筠戟戟�筠 �筠剋龜 戟逵鈞戟逵�筠戟龜�","status_unknown":"�筠龜鈞勻筠��戟�橘 ��逵���","status_attacking":"��逵克棘勻逵戟棘","status_waiting_cycle":"�菌龜畇逵戟龜筠","not_loaded":"�筠 鈞逵均��菌筠戟棘.","ignored_targets":"�均戟棘�龜��筠劇�筠 �筠剋龜","no_ignored_targets":"�龜�筠均棘 戟筠 龜均戟棘�龜�棘勻逵��","included_targets":"�克剋��筠戟戟�筠 �筠剋龜","no_included_targets":"�筠� 勻克剋��筠戟戟��","farmer_villages":"��逵閨筠菌 畇筠�筠勻筠戟�","no_farmer_villages":"�筠� 畇筠�筠勻筠戟� 畇剋� 均�逵閨筠菌逵","last_status":"�棘�剋筠畇戟龜橘 ��逵���","attacking":"��逵克棘勻逵戟棘.","paused":"��龜棘��逵戟棘勻剋筠戟.","command_limit":"�棘��龜均戟�� 剋龜劇龜� 勻 50 逵�逵克, 畇棘菌畇龜�筠�� 勻棘鈞勻�逵�筠戟龜�.","last_attack":"�棘�剋筠畇戟�� 逵�逵克逵","village_switch":"�筠�筠�棘畇 克 畇筠�筠勻戟龜","no_preset":"�筠� 畇棘���極戟�� �逵閨剋棘戟棘勻.","no_selected_village":"�筠� 畇棘���極戟�� 畇筠�筠勻筠戟�.","no_units":"�筠� 畇棘���極戟�� 勻棘橘�克 勻 畇筠�筠勻戟龜, 畇棘菌畇龜�筠�� 勻棘鈞勻�逵�筠戟龜�.","no_units_no_commands":"�龜 勻 棘畇戟棘橘 龜鈞 畇筠�筠勻筠戟� 戟筠� 勻棘鈞勻�逵�逵��龜��� 勻棘橘�克.","no_villages":"�筠� 畇棘���極戟�� 畇筠�筠勻筠戟�, 畇棘菌畇龜�筠�� 勻棘鈞勻�逵�筠戟龜� 勻棘橘�克.","preset_first":"叫��逵戟棘勻龜�筠 勻戟逵�逵剋筠 �逵閨剋棘戟!","selected_village":"��閨�逵戟戟逵� 畇筠�筠勻戟�","loading_targets":"�逵均��鈞克逵 �筠剋筠橘...","checking_targets":"��棘勻筠�克逵 �筠剋筠橘...","restarting_commands":"�筠�筠鈞逵極��克 克棘劇逵戟畇...","ignored_village":"畇棘閨逵勻龜�� 勻 龜均戟棘�龜��筠劇棘筠","included_village":"畇棘閨逵勻龜�� 勻棘 勻克剋��筠戟戟棘筠","ignored_village_removed":"�畇逵剋龜�� 龜鈞 龜均戟棘�龜��筠劇棘均棘","included_village_removed":"�畇逵剋龜�� 龜鈞 勻克剋��筠戟戟棘均棘","priority_target":"畇棘閨逵勻龜�� 勻 極�龜棘�龜�筠�戟棘筠.","analyse_targets":"�戟逵剋龜鈞 �筠剋筠橘.","step_cycle_restart":"�筠�筠鈞逵極��克 �龜克剋逵 克棘劇逵戟畇..","step_cycle_end":"�筠�筠�筠戟� 畇筠�筠勻筠戟� 棘克棘戟�筠戟, 畇棘菌畇龜�筠�� �剋筠畇���筠均棘 克��均逵.","step_cycle_end_no_villages":"�筠� 畇棘���極戟�� 畇筠�筠勻筠戟� 畇剋� ��逵��逵 �龜克剋逵.","step_cycle_next":"�筠�筠�筠戟� 畇筠�筠勻筠戟� 鈞逵克棘戟�龜剋��, �剋筠畇���龜橘 �龜克剋: %d.","step_cycle_next_no_villages":"No village available to start the cycle, next cycle: %d.","full_storage":"The storage of the village is full.","farm_stopped":"FarmOverflow stopped.","farm_started":"FarmOverflow started.","groups_presets":"Groups & presets","presets":"Attack with the presets","group_ignored":"Ignore villages from group","group_include":"Include villages from groups","group_only":"Only attack with villages from groups","attack_interval":"Interval between attacks","preserve_command_slots":"Preserve command slots","target_single_attack":"Allow targets to receive only one attack per village","target_multiple_farmers":"Allow targets to receive attacks from multiple villages","farmer_cycle_interval":"Interval between farmer cycles (minutes)","ignore_on_loss":"Ignore target that cause loss","ignore_full_storage":"Do not farm with villages with full storage","step_cycle_header":"Step Cycle Settings","step_cycle":"Enable Step Cycle","step_cycle_notifs":"Cycle notifications","target_filters":"Target Filters","min_distance":"Targets minimum distance","max_distance":"Targets maximum distance","min_points":"Targets minimum points","max_points":"Targets maximum points","max_travel_time":"Maximum travel time (minutes)","logs_limit":"Maximum amount of log entries","event_attack":"Show task logs of attacks","event_village_change":"Show task logs of village's changes","event_priority_add":"Show task logs of priority targets","event_ignored_village":"Show task logs of ignored villages","settings_saved":"Settings saved!","misc":"Miscellaneous","attack":"attack","no_logs":"No logs registered","clear_logs":"Clear logs","reseted_logs":"Registered logs reseted.","date_added":"Date added"}}})
        farmOverflow.init()
        farmOverflowInterface()
    }, ['map'])
})

define('two/minimap', [
    'two/minimap/types/actions',
    'two/minimap/settings',
    'two/minimap/settings/map',
    'two/minimap/settings/updates',
    'two/utils',
    'two/ready',
    'two/Settings',
    'queues/EventQueue',
    'Lockr',
    'struct/MapData',
    'helper/mapconvert',
    'cdn',
    'conf/colors',
    'conf/colorGroups',
    'conf/conf',
    'version'
], function (
    ACTION_TYPES,
    SETTINGS,
    SETTINGS_MAP,
    UPDATES,
    utils,
    ready,
    Settings,
    eventQueue,
    Lockr,
    mapData,
    mapconvert,
    cdn,
    colors,
    colorGroups,
    conf,
    gameVersion
) {
    let enableRendering = false
    let highlights = {}
    let villageSize = 5
    let villageMargin = 1
    let villageBlock = villageSize + villageMargin
    const rhex = /^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$/
    let cache = {
        village: {},
        character: {},
        tribe: {}
    }
    // Cached villages from previous loaded maps.
    // Used to draw the "ghost" villages on the map.
    let cachedVillages = {}
    // Data of the village that the user is hovering on the minimap.
    let hoverVillage = null
    let $viewport
    let $viewportContext
    let $viewportCache
    let $viewportCacheContext
    let $cross
    let $crossContext
    // Game main canvas map.
    // Used to calculate the position cross position based on canvas size.
    let $map
    let $player
    let $tribeRelations
    let selectedVillage
    let currentPosition = {}
    let frameSize = {}
    let dataView
    let settings = {}
    const STORAGE_KEYS = {
        CACHE_VILLAGES: 'minimap_cache_villages',
        SETTINGS: 'minimap_settings'
    }
    const colorService = injector.get('colorService')
    let allowJump = true
    let allowMove = false
    let dragStart = {}

    /**
     * Calcule the coords from clicked position in the canvas.
     *
     * @param {Object} event - Canvas click event.
     * @return {Object} X and Y coordinates.
     */
    const getCoords = function (event) {
        const villageOffsetX = minimap.getVillageAxisOffset()
        let rawX = Math.ceil(currentPosition.x + event.offsetX)
        let rawY = Math.ceil(currentPosition.y + event.offsetY)
        const adjustLine = Math.floor((rawY / villageBlock) % 2)

        if (adjustLine % 2) {
            rawX -= villageOffsetX
        }

        rawX -= rawX % villageBlock
        rawY -= rawY % villageBlock

        return {
            x: Math.ceil((rawX - frameSize.x / 2) / villageBlock),
            y: Math.ceil((rawY - frameSize.y / 2) / villageBlock)
        }
    }

    /**
     * Convert pixel wide map position to coords
     *
     * @param {Number} x - X pixel position.
     * @param {Number} y - Y pixel position.
     * @return {Object} Y and Y coordinates.
     */
    const pixel2Tiles = function (x, y) {
        return {
            x: (x / conf.TILESIZE.x),
            y: (y / conf.TILESIZE.y / conf.TILESIZE.off)
        }
    }

    /**
     * Calculate the coords based on zoom.
     *
     * @param {Array[x, y, canvasW, canvasH]} rect - Coords and canvas size.
     * @param {Number} zoom - Current zoom used to display the game original map.
     * @return {Array} Calculated coords.
     */
    const convert = function (rect, zoom) {
        zoom = 1 / (zoom || 1)

        const xy = pixel2Tiles(rect[0] * zoom, rect[1] * zoom)
        const wh = pixel2Tiles(rect[2] * zoom, rect[3] * zoom)

        return [
            xy.x - 1,
            xy.y - 1,
            (wh.x + 3) || 1,
            (wh.y + 3) || 1
        ]
    }

    /**
     * @param {Array} villages
     * @param {String=} _color - Force the village to use the
     *   specified color.
     */
    const drawVillages = function (villages, _color) {
        const pid = $player.getId()
        const tid = $player.getTribeId()
        const villageOffsetX = minimap.getVillageAxisOffset()
        const villageColors = $player.getVillagesColors()

        for (let i = 0; i < villages.length; i++) {
            let v = villages[i]
            let x
            let y
            let color

            // meta village
            if (v.id < 0) {
                continue
            }

            if (_color) {
                color = _color

                x = v[0] * villageBlock
                y = v[1] * villageBlock

                if (v[1] % 2) {
                    x += villageOffsetX
                }
            } else {
                x = v.x * villageBlock
                y = v.y * villageBlock

                if (v.y % 2) {
                    x += villageOffsetX
                }

                if (settings.get(SETTINGS.SHOW_ONLY_CUSTOM_HIGHLIGHTS)) {
                    if (v.character_id in highlights.character) {
                        color = highlights.character[v.character_id]
                    } else if (v.tribe_id in highlights.tribe) {
                        color = highlights.tribe[v.tribe_id]
                    } else {
                        continue
                    }
                } else {
                    if (v.character_id === null) {
                        if (!settings.get(SETTINGS.SHOW_BARBARIANS)) {
                            continue
                        }

                        color = villageColors.barbarian
                    } else {
                        if (v.character_id === pid) {
                            if (v.id === selectedVillage.getId() && settings.get(SETTINGS.HIGHLIGHT_SELECTED)) {
                                color = villageColors.selected
                            } else if (v.character_id in highlights.character) {
                                color = highlights.character[v.character_id]
                            } else if (settings.get(SETTINGS.HIGHLIGHT_OWN)) {
                                color = villageColors.player
                            } else {
                                color = villageColors.ugly
                            }
                        } else {
                            if (v.character_id in highlights.character) {
                                color = highlights.character[v.character_id]
                            } else if (v.tribe_id in highlights.tribe) {
                                color = highlights.tribe[v.tribe_id]
                            } else if (tid && tid === v.tribe_id && settings.get(SETTINGS.HIGHLIGHT_DIPLOMACY)) {
                                color = villageColors.tribe
                            } else if ($tribeRelations && settings.get(SETTINGS.HIGHLIGHT_DIPLOMACY)) {
                                if ($tribeRelations.isAlly(v.tribe_id)) {
                                    color = villageColors.ally
                                } else if ($tribeRelations.isEnemy(v.tribe_id)) {
                                    color = villageColors.enemy
                                } else if ($tribeRelations.isNAP(v.tribe_id)) {
                                    color = villageColors.friendly
                                } else {
                                    color = villageColors.ugly
                                }
                            } else {
                                color = villageColors.ugly
                            }
                        }
                    }
                }
            }

            $viewportCacheContext.fillStyle = color
            $viewportCacheContext.fillRect(x, y, villageSize, villageSize)
        }
    }

    const drawGrid = function () {
        const binUrl = cdn.getPath(conf.getMapPath())
        const villageOffsetX = Math.round(villageBlock / 2)

        utils.xhrGet(binUrl, function (bin) {
            dataView = new DataView(bin)

            for (let x = 1; x < 999; x++) {
                for (let y = 1; y < 999; y++) {
                    let tile = mapconvert.toTile(dataView, x, y)

                    // is border
                    if (tile.key.b) {
                        // is continental border
                        if (tile.key.c) {
                            $viewportCacheContext.fillStyle = settings.get(SETTINGS.COLOR_CONTINENT)
                            $viewportCacheContext.fillRect(x * villageBlock + villageOffsetX - 1, y * villageBlock + villageOffsetX - 1, 3, 1)
                            $viewportCacheContext.fillRect(x * villageBlock + villageOffsetX, y * villageBlock + villageOffsetX - 2, 1, 3)
                        } else {
                            $viewportCacheContext.fillStyle = settings.get(SETTINGS.COLOR_PROVINCE)
                            $viewportCacheContext.fillRect(x * villageBlock + villageOffsetX, y * villageBlock + villageOffsetX - 1, 1, 1)
                        }
                    }
                }
            }
        }, 'arraybuffer')
    }

    const drawLoadedVillages = function () {
        drawVillages(mapData.getTowns())
    }

    const drawCachedVillages = function () {
        const villageOffsetX = minimap.getVillageAxisOffset()

        for (let x in cachedVillages) {
            for (let i = 0; i < cachedVillages[x].length; i++) {
                let y = cachedVillages[x][i]
                let xx = x * villageBlock
                let yy = y * villageBlock

                if (y % 2) {
                    xx += villageOffsetX
                }

                $viewportCacheContext.fillStyle = settings.get(SETTINGS.COLOR_GHOST)
                $viewportCacheContext.fillRect(xx, yy, villageSize, villageSize)
            }
        }
    }

    /**
     * @param {Object} pos - Minimap current position plus center of canvas.
     */
    const drawViewport = function (pos) {
        $viewportContext.drawImage($viewportCache, -pos.x, -pos.y)
    }

    const clearViewport = function () {
        $viewportContext.clearRect(0, 0, $viewport.width, $viewport.height)
    }

    /**
     * @param {Object} pos - Minimap current position plus center of canvas.
     */
    const drawCross = function (pos) {
        const mapPosition = minimap.getMapPosition()
        const x = ((mapPosition[0] + mapPosition[2] - 2) * villageBlock) - pos.x
        const y = ((mapPosition[1] + mapPosition[3] - 2) * villageBlock) - pos.y

        $crossContext.fillStyle = settings.get(SETTINGS.COLOR_CROSS)
        $crossContext.fillRect(x | 0, 0, 1, lineSize)
        $crossContext.fillRect(0, y | 0, lineSize, 1)
    }

    const clearCross = function () {
        $crossContext.clearRect(0, 0, $cross.width, $cross.height)
    }

    const renderStep = function () {
        if(enableRendering) {
            const pos =  {
                x: currentPosition.x - (frameSize.x / 2),
                y: currentPosition.y - (frameSize.y / 2)
            }

            clearViewport()
            clearCross()

            drawViewport(pos)

            if (settings.get(SETTINGS.SHOW_CROSS)) {
                drawCross(pos)
            }
        }

        window.requestAnimationFrame(renderStep)
    }

    /**
     * @param {Number} sectors - Amount of sectors to be loaded,
     *   each sector has a size of 25x25 fields.
     * @param {Number=} x Optional load center X
     * @param {Number=} y Optional load center Y
     */
    const preloadSectors = function (sectors, _x, _y) {
        const size = sectors * 25
        const x = (_x || selectedVillage.getX()) - (size / 2)
        const y = (_y || selectedVillage.getY()) - (size / 2)

        mapData.loadTownDataAsync(x, y, size, size, noop)
    }

    const cacheVillages = function (villages) {
        for (let i = 0; i < villages.length; i++) {
            let v = villages[i]

            // meta village
            if (v.id < 0) {
                continue
            }

            if (!(v.x in cache.village)) {
                cache.village[v.x] = {}
            }

            if (!(v.x in cachedVillages)) {
                cachedVillages[v.x] = []
            }

            cache.village[v.x][v.y] = v.character_id || 0
            cachedVillages[v.x].push(v.y)

            if (v.character_id) {
                if (v.character_id in cache.character) {
                    cache.character[v.character_id].push([v.x, v.y])
                } else {
                    cache.character[v.character_id] = [[v.x, v.y]]
                }

                if (v.tribe_id) {
                    if (v.tribe_id in cache.tribe) {
                        cache.tribe[v.tribe_id].push(v.character_id)
                    } else {
                        cache.tribe[v.tribe_id] = [v.character_id]
                    }
                }
            }
        }

        Lockr.set(STORAGE_KEYS.CACHE_VILLAGES, cachedVillages)
    }

    const onHoverVillage = function (coords, event) {
        if (hoverVillage) {
            if (hoverVillage.x === coords.x && hoverVillage.y === coords.y) {
                return false
            } else {
                onBlurVillage()
            }
        }

        eventQueue.trigger(eventTypeProvider.MINIMAP_VILLAGE_HOVER, {
            village: mapData.getTownAt(coords.x, coords.y),
            event: event
        })

        hoverVillage = { x: coords.x, y: coords.y }
        const pid = cache.village[coords.x][coords.y]

        if (pid) {
            highlightVillages(cache.character[pid])
        } else {
            highlightVillages([[coords.x, coords.y]])
        }
    }

    const onBlurVillage = function () {
        if (!hoverVillage) {
            return false
        }

        const pid = cache.village[hoverVillage.x][hoverVillage.y]

        if (pid) {
            unhighlightVillages(cache.character[pid])
        } else {
            unhighlightVillages([[hoverVillage.x, hoverVillage.y]])
        }

        hoverVillage = false
        eventQueue.trigger(eventTypeProvider.MINIMAP_VILLAGE_BLUR)
    }

    const highlightVillages = function (villages) {
        drawVillages(villages, settings.get(SETTINGS.COLOR_QUICK_HIGHLIGHT))
    }

    const unhighlightVillages = function (villages) {
        let _villages = []

        for (let i = 0; i < villages.length; i++) {
            _villages.push(mapData.getTownAt(villages[i][0], villages[i][1]))
        }

        drawVillages(_villages)
    }

    const quickHighlight = function (coords) {
        const village = mapData.getTownAt(coords.x, coords.y)
        const action = settings.get(SETTINGS.RIGHT_CLICK_ACTION)
        let data = {}

        if (!village) {
            return false
        }

        switch (action) {
        case ACTION_TYPES.HIGHLIGHT_PLAYER:
            if (!village.character_id) {
                return false
            }

            data.type = 'character'
            data.id = village.character_id

            break
        case ACTION_TYPES.HIGHLIGHT_TRIBE:
            if (!village.tribe_id) {
                return false
            }

            data.type = 'tribe'
            data.id = village.tribe_id

            break
        }

        minimap.addHighlight(data, '#' + colors.palette.random().random())
    }

    const eventHandlers = {
        onCrossMouseDown: function (event) {
            event.preventDefault()

            allowJump = true
            allowMove = true
            dragStart = {
                x: currentPosition.x + event.pageX,
                y: currentPosition.y + event.pageY
            }

            if (hoverVillage) {
                eventQueue.trigger(eventTypeProvider.MINIMAP_VILLAGE_CLICK, [hoverVillage, event])

                // right click
                if (event.which === 3) {
                    quickHighlight(hoverVillage)
                }
            }
        },
        onCrossMouseUp: function () {
            allowMove = false
            dragStart = {}

            if (!allowJump) {
                eventQueue.trigger(eventTypeProvider.MINIMAP_STOP_MOVE)
            }
        },
        onCrossMouseMove: function (event) {
            allowJump = false

            if (allowMove) {
                currentPosition.x = dragStart.x - event.pageX
                currentPosition.y = dragStart.y - event.pageY
                eventQueue.trigger(eventTypeProvider.MINIMAP_START_MOVE)
            }

            const coords = getCoords(event)

            if (coords.x in cache.village) {
                if (coords.y in cache.village[coords.x]) {
                    let village = mapData.getTownAt(coords.x, coords.y)

                    // ignore barbarian villages
                    if (!settings.get(SETTINGS.SHOW_BARBARIANS) && !village.character_id) {
                        return false
                    }

                    // check if the village is custom highlighted
                    if (settings.get(SETTINGS.SHOW_ONLY_CUSTOM_HIGHLIGHTS)) {
                        let highlighted = false

                        if (village.character_id in highlights.character) {
                            highlighted = true
                        } else if (village.tribe_id in highlights.tribe) {
                            highlighted = true
                        }

                        if (!highlighted) {
                            return false
                        }
                    }

                    return onHoverVillage(coords, event)
                }
            }

            onBlurVillage()
        },
        onCrossMouseLeave: function () {
            if (hoverVillage) {
                onBlurVillage()
            }

            eventQueue.trigger(eventTypeProvider.MINIMAP_MOUSE_LEAVE)
        },
        onCrossMouseClick: function (event) {
            if (!allowJump) {
                return false
            }

            const coords = getCoords(event)
            $rootScope.$broadcast(eventTypeProvider.MAP_CENTER_ON_POSITION, coords.x, coords.y, true)
        },
        onCrossMouseContext: function (event) {
            event.preventDefault()
            return false
        },
        onVillageData: function (event, data) {
            drawVillages(data.villages)
            cacheVillages(data.villages)
        },
        onHighlightChange: function () {
            highlights.tribe = colorService.getCustomColorsByGroup(colorGroups.TRIBE_COLORS) || {}
            highlights.character = colorService.getCustomColorsByGroup(colorGroups.PLAYER_COLORS) || {}

            drawLoadedVillages()
        },
        onSelectedVillageChange: function () {
            const old = {
                id: selectedVillage.getId(),
                x: selectedVillage.getX(),
                y: selectedVillage.getY()
            }

            selectedVillage = $player.getSelectedVillage()

            drawVillages([{
                character_id: $player.getId(),
                id: old.id,
                x: old.x,
                y: old.y
            }, {
                character_id: $player.getId(),
                id: selectedVillage.getId(),
                x: selectedVillage.getX(),
                y: selectedVillage.getY()
            }])
        }
    }

    let minimap = {
        SETTINGS_MAP: SETTINGS_MAP,
        ACTION_TYPES: ACTION_TYPES
    }

    minimap.setVillageSize = function (value) {
        villageSize = value
        villageBlock = villageSize + villageMargin
        lineSize = 1000 * (villageSize + villageMargin)
    }

    minimap.getVillageSize = function () {
        return villageSize
    }

    minimap.setVillageMargin = function (value) {
        villageMargin = value
        villageBlock = villageSize + villageMargin
        lineSize = 1000 * (villageSize + villageMargin)
    }

    minimap.getVillageMargin = function () {
        return villageMargin
    }

    /**
     * Get the size used by each village on minimap in pixels.
     *
     * @return {Number}
     */
    minimap.getVillageBlock = function () {
        return villageBlock
    }

    minimap.getLineSize = function () {
        return lineSize
    }

    /**
     * Get the center position of a village icon.
     *
     * @return {Number}
     */
    minimap.getVillageAxisOffset = function () {
        return Math.round(villageSize / 2)
    }

    /**
     * @param {Object} item - Highlight item.
     * @param {String} item.type - village, player or tribe
     * @param {String} item.id - village/player/tribe id
     * @param {Number=} item.x - village X coord.
     * @param {Number=} item.y - village Y coord.
     * @param {String} color - Hex color
     *
     * @return {Boolean} true if successfully added
     */
    minimap.addHighlight = function (item, color) {
        if (!item || !item.type || !item.id) {
            eventQueue.trigger(eventTypeProvider.MINIMAP_HIGHLIGHT_ADD_ERROR_NO_ENTRY)
            return false
        }

        if (!rhex.test(color)) {
            eventQueue.trigger(eventTypeProvider.MINIMAP_HIGHLIGHT_ADD_ERROR_INVALID_COLOR)
            return false
        }

        highlights[item.type][item.id] = color
        const colorGroup = item.type === 'character' ? colorGroups.PLAYER_COLORS : colorGroups.TRIBE_COLORS
        colorService.setCustomColorsByGroup(colorGroup, highlights[item.type])
        $rootScope.$broadcast(eventTypeProvider.GROUPS_VILLAGES_CHANGED)

        drawLoadedVillages()

        return true
    }

    minimap.removeHighlight = function (type, itemId) {
        if (highlights[type][itemId]) {
            delete highlights[type][itemId]
            const colorGroup = type === 'character' ? colorGroups.PLAYER_COLORS : colorGroups.TRIBE_COLORS
            colorService.setCustomColorsByGroup(colorGroup, highlights[type])
            $rootScope.$broadcast(eventTypeProvider.GROUPS_VILLAGES_CHANGED)

            drawLoadedVillages()

            return true
        }

        return false
    }

    minimap.getHighlight = function (type, item) {
        if (highlights[type].hasOwnProperty(item)) {
            return highlights[type][item]
        } else {
            return false
        }
    }

    minimap.getHighlights = function () {
        return highlights
    }

    minimap.eachHighlight = function (callback) {
        for (let type in highlights) {
            for (let id in highlights[type]) {
                callback(type, id, highlights[type][id])
            }
        }
    }

    minimap.setViewport = function (element) {
        $viewport = element
        $viewport.style.background = settings.get(SETTINGS.COLOR_BACKGROUND)
        $viewportContext = $viewport.getContext('2d')
    }

    minimap.setCross = function (element) {
        $cross = element
        $crossContext = $cross.getContext('2d')
    }

    minimap.setCurrentPosition = function (x, y) {
        currentPosition.x = x * block + 50
        currentPosition.y = y * block + (1000 - ((document.body.clientHeight - 238) / 2)) + 50
    }

    /**
     * @return {Array}
     */
    minimap.getMapPosition = function () {
        if (!$map.width || !$map.height) {
            return false
        }

        if (gameVersion.product.major === 1 && gameVersion.product.minor < 94) {
            const view = window.twx.game.map.engine.getView()
        } else {
            const view = mapData.getMap().engine.getView()
        }

        return convert([
            -view.x,
            -view.y,
            $map.width / 2,
            $map.height / 2
        ], view.z)
    }

    minimap.getSettings = function () {
        return settings
    }

    minimap.update = function () {
        $viewport.style.background = settings.get(SETTINGS.COLOR_BACKGROUND)
        $viewportCacheContext.clearRect(0, 0, $viewportCache.width, $viewportCache.height)

        if (settings.get(SETTINGS.SHOW_DEMARCATIONS)) {
            drawGrid()
        }

        if (settings.get(SETTINGS.SHOW_GHOST_VILLAGES)) {
            drawCachedVillages()
        }

        drawLoadedVillages()
    }

    minimap.enableRendering = function () {
        enableRendering = true
    }

    minimap.disableRendering = function () {
        enableRendering = false
    }

    minimap.init = function () {
        minimap.initialized = true
        $viewportCache = document.createElement('canvas')
        $viewportCacheContext = $viewportCache.getContext('2d')

        settings = new Settings({
            settingsMap: SETTINGS_MAP,
            storageKey: STORAGE_KEYS.SETTINGS
        })

        settings.onChange(function (changes, updates) {
            if (updates[UPDATES.MINIMAP]) {
                minimap.update()
            }
        })

        highlights.tribe = colorService.getCustomColorsByGroup(colorGroups.TRIBE_COLORS) || {}
        highlights.character = colorService.getCustomColorsByGroup(colorGroups.PLAYER_COLORS) || {}
    }

    minimap.run = function () {
        ready(function () {
            $map = document.getElementById('main-canvas')
            $player = modelDataService.getSelectedCharacter()
            $tribeRelations = $player.getTribeRelations()
            cachedVillages = Lockr.get(STORAGE_KEYS.CACHE_VILLAGES, {}, true)

            currentPosition.x = 500 * villageBlock
            currentPosition.y = 500 * villageBlock

            frameSize.x = 701
            frameSize.y = 2000

            $viewport.setAttribute('width', frameSize.x)
            $viewport.setAttribute('height', frameSize.y)
            $viewportContext.imageSmoothingEnabled = false

            $viewportCache.setAttribute('width', 1000 * villageBlock)
            $viewportCache.setAttribute('height', 1000 * villageBlock)
            $viewportCache.imageSmoothingEnabled = false

            $cross.setAttribute('width', frameSize.x)
            $cross.setAttribute('height', frameSize.y)
            $crossContext.imageSmoothingEnabled = false

            selectedVillage = $player.getSelectedVillage()
            currentPosition.x = selectedVillage.getX() * villageBlock
            currentPosition.y = selectedVillage.getY() * villageBlock

            if (settings.get(SETTINGS.SHOW_DEMARCATIONS)) {
                drawGrid()
            }

            if (settings.get(SETTINGS.SHOW_GHOST_VILLAGES)) {
                drawCachedVillages()
            }

            drawLoadedVillages()
            cacheVillages(mapData.getTowns())
            renderStep()

            $cross.addEventListener('mousedown', eventHandlers.onCrossMouseDown)
            $cross.addEventListener('mouseup', eventHandlers.onCrossMouseUp)
            $cross.addEventListener('mousemove', eventHandlers.onCrossMouseMove)
            $cross.addEventListener('mouseleave', eventHandlers.onCrossMouseLeave)
            $cross.addEventListener('click', eventHandlers.onCrossMouseClick)
            $cross.addEventListener('contextmenu', eventHandlers.onCrossMouseContext)
            $rootScope.$on(eventTypeProvider.MAP_VILLAGE_DATA, eventHandlers.onVillageData)
            $rootScope.$on(eventTypeProvider.VILLAGE_SELECTED_CHANGED, eventHandlers.onSelectedVillageChange)
            $rootScope.$on(eventTypeProvider.TRIBE_RELATION_CHANGED, drawLoadedVillages)
            $rootScope.$on(eventTypeProvider.GROUPS_VILLAGES_CHANGED, eventHandlers.onHighlightChange)
        }, ['initial_village', 'tribe_relations'])
    }

    return minimap
})

define('two/minimap/events', [], function () {
    angular.extend(eventTypeProvider, {
        MINIMAP_HIGHLIGHT_ADD_ERROR_EXISTS: 'minimap_highlight_add_error_exists',
        MINIMAP_HIGHLIGHT_ADD_ERROR_NO_ENTRY: 'minimap_highlight_add_error_no_entry',
        MINIMAP_HIGHLIGHT_ADD_ERROR_INVALID_COLOR: 'minimap_highlight_add_error_invalid_color',
        MINIMAP_VILLAGE_CLICK: 'minimap_village_click',
        MINIMAP_VILLAGE_HOVER: 'minimap_village_hover',
        MINIMAP_VILLAGE_BLUR: 'minimap_village_blur',
        MINIMAP_MOUSE_LEAVE: 'minimap_mouse_leave',
        MINIMAP_START_MOVE: 'minimap_start_move',
        MINIMAP_STOP_MOVE: 'minimap_stop_move'
    })
})

define('two/minimap/ui', [
    'two/ui',
    'two/minimap',
    'two/minimap/types/actions',
    'two/minimap/settings',
    'two/minimap/settings/map',
    'two/utils',
    'two/EventScope',
    'two/Settings',
    'helper/util',
    'struct/MapData',
    'cdn',
    'conf/colors'
], function (
    interfaceOverflow,
    minimap,
    ACTION_TYPES,
    SETTINGS,
    SETTINGS_MAP,
    utils,
    EventScope,
    Settings,
    util,
    mapData,
    cdn,
    colors
) {
    let $scope
    let $button
    let $minimapCanvas
    let $crossCanvas
    let $minimapContainer
    let MapController
    let windowWrapper
    let mapWrapper
    let tooltipWrapper
    let tooltipTimeout
    let highlightNames = {
        character: {},
        tribe: {}
    }
    let settings
    const TAB_TYPES = {
        MINIMAP: 'minimap',
        HIGHLIGHTS: 'highlights',
        SETTINGS: 'settings'
    }
    const DEFAULT_TAB = TAB_TYPES.MINIMAP

    const selectTab = function (tab) {
        $scope.selectedTab = tab

        if (tab === TAB_TYPES.MINIMAP) {
            minimap.enableRendering()
        } else {
            minimap.disableRendering()
        }
    }

    const appendCanvas = function () {
        $minimapContainer = document.querySelector('#two-minimap .minimap-container')
        $minimapContainer.appendChild($minimapCanvas)
        $minimapContainer.appendChild($crossCanvas)
    }

    const getTribeData = function (data, callback) {
        socketService.emit(routeProvider.TRIBE_GET_PROFILE, {
            tribe_id: data.id
        }, callback)
    }

    const getCharacterData = function (data, callback) {
        socketService.emit(routeProvider.CHAR_GET_PROFILE, {
            character_id: data.id
        }, callback)
    }

    const getVillageData = function (data, callback) {
        mapData.loadTownDataAsync(data.x, data.y, 1, 1, callback)
    }

    const updateHighlightNames = function () {
        Object.keys($scope.highlights.character).forEach(function (id) {
            if (id in highlightNames.character) {
                return
            }

            getCharacterData({
                id: id
            }, function (data) {
                highlightNames.character[id] = data.character_name
            })
        })

        Object.keys($scope.highlights.tribe).forEach(function (id) {
            if (id in highlightNames.tribe) {
                return
            }

            getTribeData({
                id: id
            }, function (data) {
                highlightNames.tribe[id] = data.name
            })
        })
    }

    const showTooltip = function (_, data) {
        tooltipTimeout = setTimeout(function () {
            windowWrapper.appendChild(tooltipWrapper)
            tooltipWrapper.classList.remove('ng-hide')

            MapController.tt.name = data.village.name
            MapController.tt.x = data.village.x
            MapController.tt.y = data.village.y
            MapController.tt.province_name = data.village.province_name
            MapController.tt.points = data.village.points
            MapController.tt.character_name = data.village.character_name || '-'
            MapController.tt.character_points = data.village.character_points || 0
            MapController.tt.tribe_name = data.village.tribe_name || '-'
            MapController.tt.tribe_tag = data.village.tribe_tag || '-'
            MapController.tt.tribe_points = data.village.tribe_points || 0
            MapController.tt.morale = data.village.morale || 0
            MapController.tt.position = {}
            MapController.tt.position.x = data.event.pageX + 50
            MapController.tt.position.y = data.event.pageY + 50
            MapController.tt.visible = true

            const tooltipOffset = tooltipWrapper.getBoundingClientRect()
            const windowOffset = windowWrapper.getBoundingClientRect()
            const tooltipWrapperSpacerX = tooltipOffset.width + 50
            const tooltipWrapperSpacerY = tooltipOffset.height + 50

            onTop = MapController.tt.position.y + tooltipWrapperSpacerY > windowOffset.top + windowOffset.height
            onLeft = MapController.tt.position.x + tooltipWrapperSpacerX > windowOffset.width

            if (onTop) {
                MapController.tt.position.y -= 50
            }

            tooltipWrapper.classList.toggle('left', onLeft)
            tooltipWrapper.classList.toggle('top', onTop)
        }, 50)
    }

    const hideTooltip = function () {
        clearTimeout(tooltipTimeout)
        MapController.tt.visible = false
        tooltipWrapper.classList.add('ng-hide')
        mapWrapper.appendChild(tooltipWrapper)
    }

    const openColorPalette = function (inputType, colorGroup, itemId, itemColor) {
        let modalScope = $rootScope.$new()
        let selectedColor
        let hideReset = true
        let settingId

        modalScope.colorPalettes = colors.palette

        if (inputType === 'setting') {
            settingId = colorGroup
            selectedColor = settings.get(settingId)
            hideReset = false

            modalScope.submit = function () {
                $scope.settings[settingId] = '#' + modalScope.selectedColor
                modalScope.closeWindow()
            }

            modalScope.reset = function () {
                $scope.settings[settingId] = settings.getDefault(settingId)
                modalScope.closeWindow()
            }
        } else if (inputType === 'add_custom_highlight') {
            selectedColor = $scope.addHighlightColor

            modalScope.submit = function () {
                $scope.addHighlightColor = '#' + modalScope.selectedColor
                modalScope.closeWindow()
            }
        } else if (inputType === 'edit_custom_highlight') {
            selectedColor = $scope.highlights[colorGroup][itemId]

            modalScope.submit = function () {
                minimap.addHighlight({
                    id: itemId,
                    type: colorGroup
                }, '#' + modalScope.selectedColor)
                modalScope.closeWindow()
            }
        }

        modalScope.selectedColor = selectedColor.replace('#', '')
        modalScope.hasCustomColors = true
        modalScope.hideReset = hideReset

        modalScope.finishAction = function ($event, color) {
            modalScope.selectedColor = color
        }

        windowManagerService.getModal('modal_color_palette', modalScope)
    }

    const addCustomHighlight = function () {
        minimap.addHighlight($scope.selectedHighlight, $scope.addHighlightColor)
    }

    const saveSettings = function () {
        settings.setAll(settings.decode($scope.settings))
        utils.emitNotif('success', $filter('i18n')('settings_saved', $rootScope.loc.ale, 'minimap'))
    }

    const resetSettings = function () {
        let modalScope = $rootScope.$new()

        modalScope.title = $filter('i18n')('reset_confirm_title', $rootScope.loc.ale, 'minimap')
        modalScope.text = $filter('i18n')('reset_confirm_text', $rootScope.loc.ale, 'minimap')
        modalScope.submitText = $filter('i18n')('reset', $rootScope.loc.ale, 'common')
        modalScope.cancelText = $filter('i18n')('cancel', $rootScope.loc.ale, 'common')
        modalScope.showQuestionMarkIcon = true
        modalScope.switchColors = true

        modalScope.submit = function submit() {
            settings.resetAll()
            utils.emitNotif('success', $filter('i18n')('settings_reset', $rootScope.loc.ale, 'minimap'))
            modalScope.closeWindow()
        }

        modalScope.cancel = function cancel() {
            modalScope.closeWindow()
        }

        windowManagerService.getModal('modal_attention', modalScope)
    }

    const highlightsCount = function () {
        const character = Object.keys($scope.highlights.character).length
        const tribe = Object.keys($scope.highlights.tribe).length

        return character + tribe
    }

    const openProfile = function (type, itemId) {
        const handler = type === 'character'
            ? windowDisplayService.openCharacterProfile
            : windowDisplayService.openTribeProfile

        handler(itemId)
    }

    const eventHandlers = {
        addHighlightAutoCompleteSelect: function (item) {
            $scope.selectedHighlight = {
                id: item.id,
                type: item.type,
                name: item.name
            }
        },
        highlightUpdate: function (event) {
            updateHighlightNames()
        },
        highlightAddErrorExists: function (event) {
            utils.emitNotif('error', $filter('i18n')('highlight_add_error_exists', $rootScope.loc.ale, 'minimap'))
        },
        highlightAddErrorNoEntry: function (event) {
            utils.emitNotif('error', $filter('i18n')('highlight_add_error_no_entry', $rootScope.loc.ale, 'minimap'))
        },
        highlightAddErrorInvalidColor: function (event) {
            utils.emitNotif('error', $filter('i18n')('highlight_add_error_invalid_color', $rootScope.loc.ale, 'minimap'))
        },
        onMouseLeaveMinimap: function (event) {
            hideTooltip()

            $crossCanvas.dispatchEvent(new MouseEvent('mouseup', {
                view: window,
                bubbles: true,
                cancelable: true
            }))
        },
        onMouseMoveMinimap: function (event) {
            hideTooltip()
            $crossCanvas.style.cursor = 'url(' + cdn.getPath('/img/cursor/grab_pushed.png') + '), move'
        },
        onMouseStopMoveMinimap: function (event) {
            $crossCanvas.style.cursor = ''
        }
    }

    const init = function () {
        settings = minimap.getSettings()
        MapController = transferredSharedDataService.getSharedData('MapController')
        $minimapCanvas = document.createElement('canvas')
        $minimapCanvas.className = 'minimap'
        $crossCanvas = document.createElement('canvas')
        $crossCanvas.className = 'cross'

        minimap.setViewport($minimapCanvas)
        minimap.setCross($crossCanvas)

        tooltipWrapper = document.querySelector('#map-tooltip')
        windowWrapper = document.querySelector('#wrapper')
        mapWrapper = document.querySelector('#map')

        $button = interfaceOverflow.addMenuButton('Minimap', 50)
        $button.addEventListener('click', function () {
            const current = minimap.getMapPosition()

            if (!current) {
                return false
            }

            minimap.setCurrentPosition(current[0], current[1])
            buildWindow()
        })

        interfaceOverflow.addTemplate('twoverflow_minimap_window', `<div id=\"two-minimap\" class=\"win-content two-window\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<header class=\"win-head\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<h2>Minimap</h2>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<ul class=\"list-btn\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<li>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<a href=\"#\" class=\"size-34x34 btn-red icon-26x26-close\" ng-click=\"closeWindow()\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								</ul>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							</header>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<div class=\"win-main small-select\" scrollbar=\"\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<div class=\"tabs tabs-bg\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<div class=\"tabs-three-col\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.MINIMAP)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.MINIMAP}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.MINIMAP}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.MINIMAP}\">{{ 'minimap' | i18n:loc.ale:'minimap' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.HIGHLIGHTS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.HIGHLIGHTS}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.HIGHLIGHTS}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.HIGHLIGHTS}\">{{ 'highlights' | i18n:loc.ale:'minimap' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<div class=\"tab\" ng-click=\"selectTab(TAB_TYPES.SETTINGS)\" ng-class=\"{'tab-active': selectedTab == TAB_TYPES.SETTINGS}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<div class=\"tab-inner\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<div ng-class=\"{'box-border-light': selectedTab === TAB_TYPES.SETTINGS}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<a href=\"#\" ng-class=\"{'btn-icon btn-orange': selectedTab !== TAB_TYPES.SETTINGS}\">{{ 'settings' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<div ng-show=\"selectedTab === TAB_TYPES.MINIMAP\" class=\"minimap-container\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<div class=\"box-paper\" ng-class=\"{'footer': selectedTab == TAB_TYPES.SETTINGS}\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<div class=\"scroll-wrap\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<div ng-show=\"selectedTab == TAB_TYPES.HIGHLIGHTS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<h5 class=\"twx-section\">{{ 'add' | i18n:loc.ale:'minimap' }}</h5>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<table class=\"tbl-border-light tbl-striped add-highlight\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<col width=\"40%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<col width=\"4%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<col width=\"4%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<div auto-complete=\"autoComplete\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<td class=\"text-center\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<span ng-show=\"selectedHighlight\" class=\"icon-26x26-rte-{{ selectedHighlight.type }}\"/> {{ selectedHighlight.name }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('add_custom_highlight')\" ng-style=\"{'background-color': addHighlightColor }\" tooltip=\"\" tooltip-content=\"{{ 'tooltip_pick_color' | i18n:loc.ale:'minimap' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<span class=\"btn-orange icon-26x26-plus\" ng-click=\"addCustomHighlight()\" tooltip=\"\" tooltip-content=\"{{ 'add' | i18n:loc.ale:'minimap' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<h5 class=\"twx-section\">{{ TAB_TYPES.HIGHLIGHTS | i18n:loc.ale:'minimap' }}</h5>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<p class=\"text-center\" ng-show=\"!highlightsCount()\">{{ 'no_highlights' | i18n:loc.ale:'minimap' }}<table class=\"tbl-border-light tbl-striped\" ng-show=\"highlightsCount()\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<col width=\"4%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<col width=\"4%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<col width=\"4%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<tr ng-repeat=\"(id, color) in highlights.character\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<span class=\"icon-26x26-rte-character\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<span class=\"open-profile\" ng-click=\"openProfile('character', id)\">{{ highlightNames.character[id] }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('edit_custom_highlight', 'character', id)\" ng-style=\"{'background-color': color }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<a class=\"size-26x26 btn-red icon-20x20-close\" ng-click=\"removeHighlight('character', id)\" tooltip=\"\" tooltip-content=\"{{ 'remove' | i18n:loc.ale:'minimap' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<tr ng-repeat=\"(id, color) in highlights.tribe\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<span class=\"icon-26x26-rte-tribe\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<span class=\"open-profile\" ng-click=\"openProfile('tribe', id)\">{{ highlightNames.tribe[id] }}</span>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('edit_custom_highlight', 'tribe', id)\" ng-style=\"{'background-color': color }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<a class=\"size-26x26 btn-red icon-20x20-close\" ng-click=\"removeHighlight('tribe', id)\" tooltip=\"\" tooltip-content=\"{{ 'remove' | i18n:loc.ale:'minimap' }}\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<div ng-show=\"selectedTab == TAB_TYPES.SETTINGS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<table class=\"tbl-border-light tbl-striped\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<col width=\"60%\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<col width=\"56px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<th colspan=\"3\">{{ 'misc' | i18n:loc.ale:'minimap' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<td>{{ 'settings_right_click_action' | i18n:loc.ale:'minimap' }}<td colspan=\"3\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<div select=\"\" list=\"actionTypes\" selected=\"settings[SETTINGS.RIGHT_CLICK_ACTION]\" drop-down=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																													<td colspan=\"2\">{{ 'settings_show_cross' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<div switch-slider=\"\" value=\"settings[SETTINGS.SHOW_CROSS]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																<td colspan=\"2\">{{ 'settings_show_demarcations' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<div switch-slider=\"\" value=\"settings[SETTINGS.SHOW_DEMARCATIONS]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<td colspan=\"2\">{{ 'settings_show_barbarians' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<div switch-slider=\"\" value=\"settings[SETTINGS.SHOW_BARBARIANS]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<td colspan=\"2\">{{ 'settings_show_ghost_villages' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<div switch-slider=\"\" value=\"settings[SETTINGS.SHOW_GHOST_VILLAGES]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<td colspan=\"2\">{{ 'settings_show_only_custom_highlights' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<div switch-slider=\"\" value=\"settings[SETTINGS.SHOW_ONLY_CUSTOM_HIGHLIGHTS]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<td colspan=\"2\">{{ 'settings_highlight_own' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<div switch-slider=\"\" value=\"settings[SETTINGS.HIGHLIGHT_OWN]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<td colspan=\"2\">{{ 'settings_highlight_selected' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<div switch-slider=\"\" value=\"settings[SETTINGS.HIGHLIGHT_SELECTED]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<td colspan=\"2\">{{ 'settings_highlight_diplomacy' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<div switch-slider=\"\" value=\"settings[SETTINGS.HIGHLIGHT_DIPLOMACY]\" vertical=\"false\" size=\"'56x28'\" enabled=\"true\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																			<table class=\"tbl-border-light tbl-striped\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<col>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<col width=\"29px\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<th colspan=\"2\">{{ 'colors_misc' | i18n:loc.ale:'minimap' }}<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<td>{{ 'settings_colors_background' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_BACKGROUND)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_BACKGROUND] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																											<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																												<td>{{ 'settings_colors_province' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_PROVINCE)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_PROVINCE] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																														<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																															<td>{{ 'settings_colors_continent' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_CONTINENT)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_CONTINENT] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																	<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																		<td>{{ 'settings_colors_cross' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_CROSS)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_CROSS] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																				<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<td>{{ 'settings_colors_quick_highlight' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_QUICK_HIGHLIGHT)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_QUICK_HIGHLIGHT] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<tr>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<td>{{ 'settings_colors_ghost' | i18n:loc.ale:'minimap' }}<td>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																										<div class=\"color-container box-border-dark\" ng-click=\"openColorPalette('setting', SETTINGS.COLOR_GHOST)\" ng-style=\"{'background-color': settings[SETTINGS.COLOR_GHOST] }\"/>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									</table>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					</div>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																					<footer class=\"win-foot\" ng-show=\"selectedTab === TAB_TYPES.SETTINGS\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						<ul class=\"list-btn list-center\">
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							<li>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<a href=\"#\" class=\"btn-border btn-red\" ng-click=\"resetSettings()\">{{ 'reset' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								<li>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																									<a href=\"#\" class=\"btn-border btn-green\" ng-click=\"saveSettings()\">{{ 'save' | i18n:loc.ale:'common' }}</a>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																								</ul>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																							</footer>
																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																																						</div>`)
        interfaceOverflow.addStyle('#map-tooltip{z-index:1000}#two-minimap .minimap{position:absolute;left:0;top:38px;z-index:5}#two-minimap .cross{position:absolute;left:0;top:38px;z-index:6}#two-minimap .box-paper:not(.footer) .scroll-wrap{margin-bottom:40px}#two-minimap span.select-wrapper{width:100%}#two-minimap .add-highlight input{width:100%}#two-minimap .open-profile:hover{font-weight:bold;text-shadow:-1px 1px 0 #e0cc97}')
    }

    const buildWindow = function () {
        $scope = $rootScope.$new()
        $scope.SETTINGS = SETTINGS
        $scope.TAB_TYPES = TAB_TYPES
        $scope.selectedTab = DEFAULT_TAB
        $scope.selectedHighlight = false
        $scope.addHighlightColor = '#000000'
        $scope.highlights = minimap.getHighlights()
        $scope.highlightNames = highlightNames
        $scope.actionTypes = Settings.encodeList(ACTION_TYPES, {
            textObject: 'minimap',
            disabled: false
        })
        $scope.autoComplete = {
            type: ['character', 'tribe', 'village'],
            placeholder: $filter('i18n')('placeholder_search', $rootScope.loc.ale, 'minimap'),
            onEnter: eventHandlers.addHighlightAutoCompleteSelect
        }

        // functions
        $scope.selectTab = selectTab
        $scope.openColorPalette = openColorPalette
        $scope.addCustomHighlight = addCustomHighlight
        $scope.removeHighlight = minimap.removeHighlight
        $scope.saveSettings = saveSettings
        $scope.resetSettings = resetSettings
        $scope.highlightsCount = highlightsCount
        $scope.openProfile = openProfile

        settings.injectScope($scope, {
            textObject: 'minimap'
        })

        eventScope = new EventScope('twoverflow_minimap_window')
        eventScope.register(eventTypeProvider.GROUPS_VILLAGES_CHANGED, eventHandlers.highlightUpdate, true)
        eventScope.register(eventTypeProvider.MINIMAP_HIGHLIGHT_ADD_ERROR_EXISTS, eventHandlers.highlightAddErrorExists)
        eventScope.register(eventTypeProvider.MINIMAP_HIGHLIGHT_ADD_ERROR_NO_ENTRY, eventHandlers.highlightAddErrorNoEntry)
        eventScope.register(eventTypeProvider.MINIMAP_HIGHLIGHT_ADD_ERROR_INVALID_COLOR, eventHandlers.highlightAddErrorInvalidColor)
        eventScope.register(eventTypeProvider.MINIMAP_VILLAGE_HOVER, showTooltip)
        eventScope.register(eventTypeProvider.MINIMAP_VILLAGE_BLUR, hideTooltip)
        eventScope.register(eventTypeProvider.MINIMAP_MOUSE_LEAVE, eventHandlers.onMouseLeaveMinimap)
        eventScope.register(eventTypeProvider.MINIMAP_START_MOVE, eventHandlers.onMouseMoveMinimap)
        eventScope.register(eventTypeProvider.MINIMAP_STOP_MOVE, eventHandlers.onMouseStopMoveMinimap)

        windowManagerService.getScreenWithInjectedScope('!twoverflow_minimap_window', $scope)
        updateHighlightNames()
        appendCanvas()
        minimap.enableRendering()
    }

    return init
})

define('two/minimap/settings', [], function () {
    return {
        RIGHT_CLICK_ACTION: 'right_click_action',
        FLOATING_MINIMAP: 'floating_minimap',
        SHOW_CROSS: 'show_cross',
        SHOW_DEMARCATIONS: 'show_demarcations',
        SHOW_BARBARIANS: 'show_barbarians',
        SHOW_GHOST_VILLAGES: 'show_ghost_villages',
        SHOW_ONLY_CUSTOM_HIGHLIGHTS: 'show_only_custom_highlights',
        HIGHLIGHT_OWN: 'highlight_own',
        HIGHLIGHT_SELECTED: 'highlight_selected',
        HIGHLIGHT_DIPLOMACY: 'highlight_diplomacy',
        COLOR_GHOST: 'color_ghost',
        COLOR_QUICK_HIGHLIGHT: 'color_quick_highlight',
        COLOR_BACKGROUND: 'color_background',
        COLOR_PROVINCE: 'color_province',
        COLOR_CONTINENT: 'color_continent',
        COLOR_CROSS: 'color_cross'
    }
})

define('two/minimap/settings/updates', function () {
    return {
        MINIMAP: 'minimap'
    }
})

define('two/minimap/settings/map', [
    'two/minimap/settings',
    'two/minimap/types/actions',
    'two/minimap/settings/updates'
], function (
    SETTINGS,
    ACTION_TYPES,
    UPDATES
) {
    return {
        [SETTINGS.RIGHT_CLICK_ACTION]: {
            default: ACTION_TYPES.HIGHLIGHT_PLAYER,
            inputType: 'select',
            disabledOption: false
        },
        [SETTINGS.SHOW_CROSS]: {
            default: true,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.SHOW_DEMARCATIONS]: {
            default: true,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.SHOW_BARBARIANS]: {
            default: false,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.SHOW_GHOST_VILLAGES]: {
            default: false,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.SHOW_ONLY_CUSTOM_HIGHLIGHTS]: {
            default: false,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.HIGHLIGHT_OWN]: {
            default: true,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.HIGHLIGHT_SELECTED]: {
            default: true,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.HIGHLIGHT_DIPLOMACY]: {
            default: true,
            inputType: 'checkbox',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_SELECTED]: {
            default: '#ffffff',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_BARBARIAN]: {
            default: '#969696',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_PLAYER]: {
            default: '#f0c800',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_QUICK_HIGHLIGHT]: {
            default: '#ffffff',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_BACKGROUND]: {
            default: '#436213',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_PROVINCE]: {
            default: '#ffffff',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_CONTINENT]: {
            default: '#cccccc',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_CROSS]: {
            default: '#999999',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_TRIBE]: {
            default: '#0000DB',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_ALLY]: {
            default: '#00a0f4',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_ENEMY]: {
            default: '#ED1212',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_FRIENDLY]: {
            default: '#BF4DA4',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        },
        [SETTINGS.COLOR_GHOST]: {
            default: '#3E551C',
            inputType: 'color',
            updates: [UPDATES.MINIMAP]
        }
    }
})

define('two/minimap/types/actions', [], function () {
    return {
        HIGHLIGHT_PLAYER: 'highlight_player',
        HIGHLIGHT_TRIBE: 'highlight_tribe'
    }
})

require([
    'two/language',
    'two/ready',
    'two/minimap',
    'two/minimap/ui',
    'two/minimap/events',
    'two/minimap/types/actions',
    'two/minimap/settings',
    'two/minimap/settings/updates',
    'two/minimap/settings/map'
], function (
    twoLanguage,
    ready,
    minimap,
    minimapInterface
) {
    if (minimap.initialized) {
        return false
    }

    ready(function () {
        twoLanguage.add('minimap', {"de_de":{"minimap":{"minimap":"Minimap","highlights":"Farboptionen","add":"Farboption erstellen","remove":"Farboption entfernen","placeholder_search":"Suche Spieler/Stamm","highlight_add_success":"Farboption hinzugef체gt","highlight_add_error":"W채hle zuerst eine Farboption","highlight_update_success":"Farboptionen aktualisiert","highlight_remove_success":"Farboption entfernt","highlight_villages":"D철rfer","highlight_players":"Spieler","highlight_tribes":"St채mme","highlight_add_error_exists":"Farboption existiert bereits!","highlight_add_error_no_entry":"W채hle zuerst einen Spieler/Stamm aus!","highlight_add_error_invalid_color":"Ung체ltige Farbe!","village":"Dorf","player":"Spieler","tribe":"Stamm","color":"Wert (Hex)","tooltip_pick_color":"Eine Farbe ausw채hlen","misc":"Weitere Einstellungen","colors_misc":"Sonstige Farben","colors_diplomacy":"Preset Farbenoptionen der Stammesdiplomatie","settings_saved":"Einstellungen gespeichert!","settings_right_click_action":"Rechtsklickoption des Dorfes","highlight_village":"Dorf hervorheben","highlight_player":"Spieler hervorheben","highlight_tribe":"Stamm hervorheben","settings_show_floating_minimap":"Zeige Minimap","settings_show_cross":"Positionskreuz anzeigen","settings_show_demarcations":"Provinzgrenzen und K철nigreichsgrenzen anzeigen","settings_show_barbarians":"Zeige Barbarend철rfer","settings_show_ghost_villages":"Nicht geladene D철rfer anzeigen","settings_show_only_custom_highlights":"Nur benutzerdefinierte Farboptionen anzeigen","settings_highlight_own":"Eigene D철rfer automatisch hervorheben","settings_highlight_selected":"Ausgew채hltes Dorf automatisch hervorheben","settings_highlight_diplomacy":"Stammesdiplomatie automatisch hervorheben","settings_colors_background":"Hintergrund der Karte","settings_colors_province":"Provinzgrenze","settings_colors_continent":"K철nigreichsgrenze","settings_colors_quick_highlight":"Schnelles Hervorheben","settings_colors_tribe":"Eigener Stamm","settings_colors_player":"Eigene D철rfer","settings_colors_selected":"Ausgew채hltes Dorf","settings_colors_ghost":"Nicht geladene D철rfer","settings_colors_ally":"B체ndnis","settings_colors_pna":"NAP","settings_colors_enemy":"Feind","settings_colors_other":"Sonstige","settings_colors_barbarian":"Barbaren","settings_colors_cross":"Positionskreuz","settings_reset":"Einstellungen wurden zur체ckgesetzt","tooltip_village":"Dorf","tooltip_village_points":"Punkte des Dorfes","tooltip_player":"Spielername","tooltip_player_points":"Spielerpunkte","tooltip_tribe":"Stamm","tooltip_tribe_points":"Punkte des Stammes","tooltip_province":"Name der Provinz","no_highlights":"Keine Farboptionen erstellt","reset_confirm_title":"Einstellungen zur체ck setzen","reset_confirm_text":"Alle Einstellungen werden auf Standardeinstellungen zur체ckgesetzt.","reset_confirm_highlights_text":"Es werden auch alle Farboptionen gel철scht."}},"en_us":{"minimap":{"minimap":"Minimap","highlights":"Highlights","add":"Add highlight","remove":"Remove highlight","placeholder_search":"Search player/tribe","highlight_add_success":"Highlight added","highlight_add_error":"Specify a highlight first","highlight_update_success":"Highlight updated","highlight_remove_success":"Highlight removed","highlight_villages":"Villages","highlight_players":"Players","highlight_tribes":"Tribes","highlight_add_error_exists":"Highlight already exists!","highlight_add_error_no_entry":"Select a player/tribe first!","highlight_add_error_invalid_color":"Invalid color!","village":"Village","player":"Player","tribe":"Tribe","color":"Color (Hex)","tooltip_pick_color":"Select a color","misc":"Miscellaneous settings","colors_misc":"Miscellaneous colors","colors_diplomacy":"Diplomacy colors","settings_saved":"Settings saved!","settings_right_click_action":"Village's right click action","highlight_village":"Highlight village","highlight_player":"Highlight player","highlight_tribe":"Highlight tribe","settings_show_floating_minimap":"Show floating minimap","settings_show_cross":"Show position cross","settings_show_demarcations":"Show province/continent demarcations","settings_show_barbarians":"Show barbarian villages","settings_show_ghost_villages":"Show non-loaded villages","settings_show_only_custom_highlights":"Show only custom highlights","settings_highlight_own":"Highlight own villages","settings_highlight_selected":"highlight selected village","settings_highlight_diplomacy":"Auto highlight tribe diplomacies","settings_colors_background":"Minimap background","settings_colors_province":"Province demarcation","settings_colors_continent":"Continent demarcation","settings_colors_quick_highlight":"Quick highlight","settings_colors_tribe":"Own tribe","settings_colors_player":"Own villages","settings_colors_selected":"Selected village","settings_colors_ghost":"Non-loaded villages","settings_colors_ally":"Ally","settings_colors_pna":"PNA","settings_colors_enemy":"Enemy","settings_colors_other":"Other","settings_colors_barbarian":"Barbarian","settings_colors_cross":"Position cross","settings_reset":"Settings reseted","tooltip_village":"Village","tooltip_village_points":"Village points","tooltip_player":"Player name","tooltip_player_points":"Player points","tooltip_tribe":"Tribe","tooltip_tribe_points":"Tribe points","tooltip_province":"Province name","no_highlights":"No highlights created","reset_confirm_title":"Reset settings","reset_confirm_text":"All settings gonna be reseted to the default settings.","reset_confirm_highlights_text":"Also, all highlights are going to be deleted."}},"pl_pl":{"minimap":{"minimap":"Minimap","highlights":"Pod힄wietlenie","add":"Dodaj pod힄wietlenie","remove":"Usu흦 pod힄wietlenie","placeholder_search":"Szukaj gracz/plemie","highlight_add_success":"Pod힄wietlenie dodane","highlight_add_error":"Najpierw sprecyzuj pod힄wietlenie","highlight_update_success":"Pod힄wietlenie zaktualizowane","highlight_remove_success":"Pod힄wietlenie usuni휌te","highlight_villages":"Wioski","highlight_players":"Gracze","highlight_tribes":"Plemiona","highlight_add_error_exists":"Pod힄wietlenie ju탉 istnieje!","highlight_add_error_no_entry":"Najpierw wybierz gracza/plemi휌!","highlight_add_error_invalid_color":"Nieprawid흢owy kolor!","village":"Wioska","player":"Gracz","tribe":"Plemi휌","color":"Kolor (Hex)","tooltip_pick_color":"Wybierz kolor","misc":"Pozosta흢e ustawienia","colors_misc":"R처탉ne kolory","colors_diplomacy":"Dyplomacja - kolory","settings_saved":"Ustawienia zapisane!","settings_right_click_action":"PPM aby wykona훶 dzia흢anie na wiosce","highlight_village":"Pod힄wietl wiosk휌","highlight_player":"Pod힄wietl gracza","highlight_tribe":"Pod힄wietl plemie","settings_show_floating_minimap":"Poka탉 ruchom훳 map휌","settings_show_cross":"Poka탉 krzy탉 pozycjonowania","settings_show_demarcations":"Poka탉 granice prowincji/kontynent처w","settings_show_barbarians":"Poka탉 wioski barbarzy흦skie","settings_show_ghost_villages":"Poka탉 nieza흢adowane wioski","settings_show_only_custom_highlights":"Poka탉 tylko w흢asne pod힄wietlenia","settings_highlight_own":"Pod힄wietl w흢asne wioski","settings_highlight_selected":"Pod힄wietl wybran훳 wiosk휌","settings_highlight_diplomacy":"Automatycznie pod힄wietl plemienn훳 dyplomacj휌","settings_colors_background":"T흢o minimapy","settings_colors_province":"Granica prowincji","settings_colors_continent":"Granica kontynentu","settings_colors_quick_highlight":"Szybkie pod힄wietlenie","settings_colors_tribe":"W흢asne plemie","settings_colors_player":"W흢asne wioski","settings_colors_selected":"Wybrana wioska","settings_colors_ghost":"Nieza흢adowana wioska","settings_colors_ally":"Sojusznik","settings_colors_pna":"PON","settings_colors_enemy":"Wr처g","settings_colors_other":"Pozosta흢e wioski graczy","settings_colors_barbarian":"Wioski barbarzy흦skie","settings_colors_cross":"Krzy탉 pozycjonowania","settings_reset":"Ustawienia zresetowane","tooltip_village":"Wioska","tooltip_village_points":"Punkty wioski","tooltip_player":"Nazwa gracza","tooltip_player_points":"Punkty gracza","tooltip_tribe":"Plemi휌","tooltip_tribe_points":"Punkty plemienia","tooltip_province":"Prowincja","no_highlights":"Brak utworzonych pod힄wietle흦","reset_confirm_title":"Resetuj ustawienia","reset_confirm_text":"Wszystkie ustawienia zostan훳 przywr처cone do domy힄lnych.","reset_confirm_highlights_text":"Jak r처wnie탉 wszystkie pod힄wietlenia zostan훳 usuni휌te."}},"pt_br":{"minimap":{"minimap":"Minimapa","highlights":"Marca챌천es","add":"Adicionar marca챌찾o","remove":"Remover marca챌찾o","placeholder_search":"Procurar jogador/tribo","highlight_add_success":"Marca챌찾o adicionada","highlight_add_error":"Especifique uma marca챌찾o primeiro","highlight_update_success":"Marca챌찾o atualizada","highlight_remove_success":"Marca챌찾o removida","highlight_villages":"Aldeias","highlight_players":"Jogadores","highlight_tribes":"Tribos","highlight_add_error_exists":"Marca챌찾o j찼 existe!","highlight_add_error_no_entry":"Selecione uma jogador/tribo primeiro!","highlight_add_error_invalid_color":"Cor inv찼lida!","village":"Aldeia","player":"Jogador","tribe":"Tribo","color":"Cor (Hex)","tooltip_pick_color":"Selecione uma cor","misc":"Configura챌천es diversas","colors_misc":"Cores diversas","colors_diplomacy":"Cores da diplomacia","settings_saved":"Configura챌천es salvas!","settings_right_click_action":"A챌찾o de clique direito na aldeia","highlight_village":"Marcar aldeia","highlight_player":"Marcar jogador","highlight_tribe":"Marcar tribo","settings_show_floating_minimap":"Mostrar minimapa flutuante","settings_show_cross":"Mostrar marca챌찾o da posi챌찾o atual","settings_show_demarcations":"Mostrar demarca챌천es das provincias/continentes","settings_show_barbarians":"Mostrar aldeias b찼rbaras","settings_show_ghost_villages":"Mostrar aldeias n찾o carregadas","settings_show_only_custom_highlights":"Mostrar apenas marca챌천es manuais","settings_highlight_own":"Marcar pr처prias aldeias","settings_highlight_selected":"Marcar aldeia selecionada","settings_highlight_diplomacy":"Marca챌찾o autom찼tica baseado na diplomacia","settings_colors_background":"Fundo do minimapa","settings_colors_province":"Demarca챌찾o da provincia","settings_colors_continent":"Demarca챌찾o do continente","settings_colors_quick_highlight":"Marca챌찾o r찼pida","settings_colors_tribe":"Pr처pria tribo","settings_colors_player":"Aldeias pr처prias","settings_colors_selected":"Aldeia selecionada","settings_colors_ghost":"Aldeias n찾o carregadas","settings_colors_ally":"Aliados","settings_colors_pna":"PNA","settings_colors_enemy":"Inimigos","settings_colors_other":"Outros","settings_colors_barbarian":"Aldeias B찼rbaras","settings_colors_cross":"Cruz da posi챌찾o","settings_reset":"Configura챌천es resetadas","tooltip_village":"Aldeia","tooltip_village_points":"Pontos da aldeia","tooltip_player":"Nome do jogador","tooltip_player_points":"Pontos do jogador","tooltip_tribe":"Tribo","tooltip_tribe_points":"Pontos da tribo","tooltip_province":"Nome da prov챠ncia","no_highlights":"Nenhuma marca챌찾o criada","reset_confirm_title":"Resetar configura챌천es","reset_confirm_text":"Todas as configura챌천es ser찾o resetas para as configura챌천es padr천es.","reset_confirm_highlights_text":"Todas marca챌천es tamb챕m ser찾o deletadas"}},"ru_ru":{"minimap":{"minimap":"�龜戟龜克逵��逵","highlights":"Highlights","add":"Add highlight","remove":"Remove highlight","placeholder_search":"Search player/tribe","highlight_add_success":"Highlight added","highlight_add_error":"Specify a highlight first","highlight_update_success":"Highlight updated","highlight_remove_success":"Highlight removed","highlight_villages":"Villages","highlight_players":"Players","highlight_tribes":"Tribes","highlight_add_error_exists":"Highlight already exists!","highlight_add_error_no_entry":"Select a player/tribe first!","highlight_add_error_invalid_color":"Invalid color!","village":"Village","player":"Player","tribe":"Tribe","color":"Color (Hex)","tooltip_pick_color":"Select a color","misc":"Miscellaneous settings","colors_misc":"Miscellaneous colors","colors_diplomacy":"Diplomacy colors","settings_saved":"Settings saved!","settings_right_click_action":"Village's right click action","highlight_village":"Highlight village","highlight_player":"Highlight player","highlight_tribe":"Highlight tribe","settings_show_floating_minimap":"Show floating minimap","settings_show_cross":"Show position cross","settings_show_demarcations":"Show province/continent demarcations","settings_show_barbarians":"Show barbarian villages","settings_show_ghost_villages":"Show non-loaded villages","settings_show_only_custom_highlights":"Show only custom highlights","settings_highlight_own":"Highlight own villages","settings_highlight_selected":"highlight selected village","settings_highlight_diplomacy":"Auto highlight tribe diplomacies","settings_colors_background":"Minimap background","settings_colors_province":"Province demarcation","settings_colors_continent":"Continent demarcation","settings_colors_quick_highlight":"Quick highlight","settings_colors_tribe":"Own tribe","settings_colors_player":"Own villages","settings_colors_selected":"Selected village","settings_colors_ghost":"Non-loaded villages","settings_colors_ally":"Ally","settings_colors_pna":"PNA","settings_colors_enemy":"Enemy","settings_colors_other":"Other","settings_colors_barbarian":"Barbarian","settings_colors_cross":"Position cross","settings_reset":"Settings reseted","tooltip_village":"Village","tooltip_village_points":"Village points","tooltip_player":"Player name","tooltip_player_points":"Player points","tooltip_tribe":"Tribe","tooltip_tribe_points":"Tribe points","tooltip_province":"Province name","no_highlights":"No highlights created","reset_confirm_title":"Reset settings","reset_confirm_text":"All settings gonna be reseted to the default settings.","reset_confirm_highlights_text":"Also, all highlights are going to be deleted."}}})
        minimap.init()
        minimapInterface()
        minimap.run()
    })
})

define('two/ui', [
    'conf/conf',
    'conf/cdn'
], function (
    conf,
    cdnConf
) {
    let interfaceOverflow = {}
    let templates = {}
    let initialized = false
    let containerCreated = false

    let $head = document.querySelector('head')
    let $wrapper = document.querySelector('#wrapper')
    let httpService = injector.get('httpService')
    let templateManagerService = injector.get('templateManagerService')
    let $templateCache = injector.get('$templateCache')

    let $container
    let $menu
    let $mainButton

    const buildMenuContainer = function () {
        if (containerCreated) {
            return true
        }

        $container = document.createElement('div')
        $container.className = 'two-menu-container'
        $wrapper.appendChild($container)

        $mainButton = document.createElement('div')
        $mainButton.className = 'two-main-button'
        $container.appendChild($mainButton)

        $menu = document.createElement('div')
        $menu.className = 'two-menu'
        $container.appendChild($menu)

        containerCreated = true
    }

    templateManagerService.load = function (templateName, onSuccess, opt_onError) {
        let path

        const success = function (data, status, headers, config) {
            $templateCache.put(path.substr(1), data)

            if (angular.isFunction(onSuccess)) {
                onSuccess(data, status, headers, config)
            }

            if (!$rootScope.$$phase) {
                $rootScope.$apply()
            }
        }

        const error = function (data, status, headers, config) {
            if (angular.isFunction(opt_onError)) {
                opt_onError(data, status, headers, config)
            }
        }

        if (0 !== templateName.indexOf('!')) {
            path = conf.TEMPLATE_PATH_EXT.join(templateName)
        } else {
            path = templateName.substr(1)
        }

        if ($templateCache.get(path.substr(1))) {
            success($templateCache.get(path.substr(1)), 304)
        } else {
            if (cdnConf.versionMap[path]) {
                httpService.get(path, success, error)
            } else {
                success(templates[path], 304)
            }
        }
    }

    interfaceOverflow.init = function () {
        if (initialized) {
            return false
        }

        initialized = true
        interfaceOverflow.addStyle('.two-window a.select-handler{-webkit-box-shadow:none;box-shadow:none}.two-window .small-select a.select-handler{height:22px;line-height:22px}.two-window .small-select a.select-button{height:22px}.two-window input::placeholder{color:rgba(255,243,208,0.7)}.two-window .green{color:#07770b}.two-window .red{color:#770707}.two-window .blue{color:#074677}#toolbar-left{height:calc(100% - 273px) !important;top:165px !important}.two-menu-container{position:absolute;top:84px;left:0;width:90px;z-index:10}.two-menu-container:hover .two-main-button{background-position:0 -75px}.two-menu-container:hover .two-menu{opacity:1;visibility:visible;transition:opacity .1s ease-in-out}.two-main-button{left:0;width:75px;height:75px;background-image:url(https://i.imgur.com/J0JFhf4.png);background-position:0 0;z-index:10}.two-menu{position:absolute;top:8px;left:85px;width:120px;display:flex;flex-flow:wrap;visibility:hidden;opacity:0;z-index:10}.two-menu .button{height:24px;padding:0 15px;line-height:22px;margin-bottom:8px}#wrapper.window-open .two-menu-container{left:713px}#wrapper.window-open.window-fullsize .two-menu-container{display:none !important}')
    }

    interfaceOverflow.addTemplate = function (path, data) {
        templates[path] = data
    }

    interfaceOverflow.addStyle = function (styles) {
        let $style = document.createElement('style')
        $style.type = 'text/css'
        $style.appendChild(document.createTextNode(styles))
        $head.appendChild($style)
    }

    interfaceOverflow.addMenuButton = function (label, order, _tooltip) {
        buildMenuContainer()

        let $button = document.createElement('div')
        $button.className = 'btn-border btn-green button'
        $button.innerHTML = label
        $button.style.order = order
        $menu.appendChild($button)

        if (typeof _tooltip === 'string') {
            $button.addEventListener('mouseenter', function (event) {
                $rootScope.$broadcast(eventTypeProvider.TOOLTIP_SHOW, 'twoverflow-tooltip', _tooltip, true, event)
            })

            $button.addEventListener('mouseleave', function () {
                $rootScope.$broadcast(eventTypeProvider.TOOLTIP_HIDE, 'twoverflow-tooltip')
            })
        }

        return $menu.appendChild($button)
    }

    interfaceOverflow.isInitialized = function () {
        return initialized
    }

    return interfaceOverflow
})

require([
    'two/ui'
], function (interfaceOverflow) {
    if (interfaceOverflow.isInitialized()) {
        return false
    }

    interfaceOverflow.init()
})

})(this)
