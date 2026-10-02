# Graph Report - qgc-focus  (2026-10-02)

## Corpus Check
- 217 files · ~138,791 words
- Verdict: corpus is large enough that graph structure adds value.
- Unclassified: 104 file(s) not represented in the graph (top: .qml 104)

## Summary
- 6707 nodes · 12345 edges · 280 communities (183 shown, 97 thin omitted)
- Extraction: 89% EXTRACTED · 11% INFERRED · 0% AMBIGUOUS · INFERRED: 1332 edges (avg confidence: 0.85)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- Community 0
- Community 1
- Community 2
- Community 3
- Community 4
- Community 5
- Community 6
- Community 7
- Community 8
- Community 9
- Community 10
- Community 11
- Community 12
- Community 13
- Community 15
- Community 16
- Community 17
- Community 18
- Community 19
- Community 20
- Community 21
- Community 22
- Community 23
- Community 24
- Community 25
- Community 26
- Community 27
- Community 28
- Community 29
- Community 30
- Community 31
- Community 32
- Community 33
- Community 34
- Community 35
- Community 36
- Community 37
- Community 38
- Community 40
- Community 41
- Community 42
- Community 43
- Community 45
- Community 46
- Community 47
- Community 48
- Community 49
- Community 50
- Community 51
- Community 52
- Community 53
- Community 55
- Community 56
- Community 57
- Community 58
- Community 59
- Community 60
- Community 61
- Community 62
- Community 63
- Community 64
- Community 65
- Community 66
- Community 67
- Community 68
- Community 69
- Community 70
- Community 71
- Community 72
- Community 73
- Community 74
- Community 75
- Community 76
- Community 77
- Community 78
- Community 79
- Community 80
- Community 81
- Community 82
- Community 83
- Community 84
- Community 85
- Community 86
- Community 87
- Community 88
- Community 89
- Community 90
- Community 91
- Community 92
- Community 93
- Community 94
- Community 95
- Community 97
- Community 98
- Community 99
- Community 100
- Community 101
- Community 102
- Community 104
- Community 106
- Community 107
- Community 108
- Community 109
- Community 110
- Community 111
- Community 112
- Community 113
- Community 114
- Community 115
- Community 116
- Community 117
- Community 118
- Community 119
- Community 120
- Community 121
- Community 122
- Community 123
- Community 124
- Community 125
- Community 126
- Community 127
- Community 128
- Community 129
- Community 130
- Community 131
- Community 132
- Community 133
- Community 134
- Community 135
- Community 136
- Community 137
- Community 138
- Community 139
- Community 140
- Community 141
- Community 143
- Community 144
- Community 145
- Community 146
- Community 147
- Community 148
- Community 150
- Community 151
- Community 152
- Community 153
- Community 154
- Community 155
- Community 156
- Community 157
- Community 158
- Community 159
- Community 160
- Community 161
- Community 162
- Community 164
- Community 165
- Community 166
- Community 167
- Community 168
- Community 169
- Community 170
- Community 171
- Community 172
- Community 173
- Community 174
- Community 175
- Community 176
- Community 178
- Community 179
- Community 180
- Community 182
- Community 183
- Community 184
- Community 185
- Community 186
- Community 188
- Community 189
- Community 190
- Community 191
- Community 192
- Community 193
- Community 194
- Community 195
- Community 196
- Community 198
- Community 200
- Community 201
- Community 202
- Community 203
- Community 205
- Community 206
- Community 208
- Community 209
- Community 210
- Community 211
- Community 213
- Community 215
- Community 216
- Community 217
- Community 218
- Community 219
- Community 220
- Community 221
- Community 224
- Community 225
- Community 226
- Community 227
- Community 228
- Community 229
- Community 230
- Community 233
- Community 234
- Community 235
- Community 237
- Community 238
- Community 239
- Community 240
- Community 241
- Community 243
- Community 247
- Community 248
- Community 249
- Community 250
- Community 253
- Community 254
- Community 256
- Community 257
- Community 258
- Community 259
- Community 260
- Community 261
- Community 262
- Community 267
- Community 269
- Community 270
- Community 271
- Community 274
- Community 276

## God Nodes (most connected - your core abstractions)
1. `Vehicle` - 579 edges
2. `RemoteControlCalibrationController` - 222 edges
3. `AUX_FUNC` - 180 edges
4. `FirmwareUpgradeController` - 133 edges
5. `BluetoothConfiguration` - 116 edges
6. `MAVLinkLogManager` - 100 edges
7. `FirmwarePlugin` - 99 edges
8. `FTPManager` - 91 edges
9. `APMFirmwarePlugin` - 88 edges
10. `VehicleFactGroup` - 83 edges

## Surprising Connections (you probably didn't know these)
- `LinkManager::_addSerialAutoConnectLink()` --references--> `SerialConfiguration`  [INFERRED]
  src/Comms/LinkManager.cc → src/Comms/SerialLink.h
- `getChannelStatus()` --calls--> `mavlink_get_channel_status()`  [INFERRED]
  src/MAVLink/QGCMAVLink.h → src/MAVLink/QGCMAVLink.cc
- `_channelSigningPtr()` --calls--> `mavlink_get_channel_status()`  [INFERRED]
  src/MAVLink/Signing/MAVLinkSigning.cc → src/MAVLink/QGCMAVLink.cc
- `encodeSetupSigning()` --calls--> `mavlink_get_channel_status()`  [INFERRED]
  src/MAVLink/Signing/MAVLinkSigning.cc → src/MAVLink/QGCMAVLink.cc
- `signingStreamCount()` --calls--> `mavlink_get_channel_status()`  [INFERRED]
  src/MAVLink/Signing/MAVLinkSigning.cc → src/MAVLink/QGCMAVLink.cc

## Import Cycles
- None detected.

## Communities (280 total, 97 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.01
Nodes (90): Vehicle, _allLinksRemovedSent, _altitudeTuningOffset, _base_mode, _batteryFactGroupListModel, _cameraImageCapturedMessageAvailable, _cameraManager, _capabilityBitsKnown (+82 more)

### Community 1 - "Community 1"
Cohesion: 0.01
Nodes (180): AUX_FUNC, ACRO, ACRO_TRAINER, AHRS_AUTO_TRIM, AHRS_TYPE, AIRBRAKE, AIRMODE, ALTHOLD (+172 more)

### Community 2 - "Community 2"
Cohesion: 0.02
Nodes (3): FactGroup, Vehicle::healthAndArmingCheckReport(), Vehicle::links()

### Community 3 - "Community 3"
Cohesion: 0.02
Nodes (39): RemoteControlCalibrationController, _bothStickDisplayPositionThrottleCenteredMap, _bothStickDisplayPositionThrottleDownMap, _calCenterPoint, _calDefaultMaxValue, _calDefaultMinValue, _calMoveDelta, _calRoughCenterDelta (+31 more)

### Community 4 - "Community 4"
Cohesion: 0.04
Nodes (20): LinkManager, _autoConnectSettings, _autoconnectUpdateTimerMSecs, _configUpdateSuspended, _configurationsLoaded, _connectionsSuspended, _connectionsSuspendedReason, _defaultUDPLinkName (+12 more)

### Community 5 - "Community 5"
Cohesion: 0.03
Nodes (55): FirmwareUpgradeController, _activeDownloader, _apmBoardDescriptionReplaceText, _apmChibiOSSetting, _apmFirmwareNames, _apmFirmwareNamesBestIndex, _apmFirmwareUrls, _apmVehicleTypeFromCurrentVersionList (+47 more)

### Community 6 - "Community 6"
Cohesion: 0.05
Nodes (17): FTPController, _extractionJob, _extractionOutputDir, FTPController::FTPController(), _ftpManager, _operation, QML_ELEMENT, _vehicle (+9 more)

### Community 7 - "Community 7"
Cohesion: 0.03
Nodes (57): Actuators, AutoPilotPlugin, Autotune, BatteryFactGroupListModel, ComponentInformationManager, EscStatusFactGroupListModel, FTPManager, GeoFenceManager (+49 more)

### Community 8 - "Community 8"
Cohesion: 0.06
Nodes (22): MAVLinkSigningKey, _keyBytes, MAVLinkSigningKey::MAVLinkSigningKey(), _name, QML_ELEMENT, MAVLinkSigningKeys, _keyIndex, kKeySubgroup (+14 more)

### Community 9 - "Community 9"
Cohesion: 0.05
Nodes (7): FirmwarePlugin, _flightModeList, _modeEnumToString, _modeIndicatorList, public, _toolIndicatorList, Vehicle

### Community 10 - "Community 10"
Cohesion: 0.05
Nodes (22): RadioStatusFactGroup, _fixedFact, _lNoiseFact, _lrssiFact, RadioStatusFactGroup::RadioStatusFactGroup(), _rNoiseFact, _rrssiFact, _rxErrorsFact (+14 more)

### Community 11 - "Community 11"
Cohesion: 0.04
Nodes (14): LinkConfiguration, _autoConnect, _connectedTimer, _dynamic, _forwarding, _highLatency, LinkConfiguration::LinkConfiguration(), _nextReconnect (+6 more)

### Community 12 - "Community 12"
Cohesion: 0.06
Nodes (19): BluetoothBleWorker, BLE_MAX_PACKET_SIZE, BLE_MIN_PACKET_SIZE, _bleWriteInProgress, _bleWriteQueue, _controller, _currentBleWrite, DEFAULT_ATT_MTU (+11 more)

### Community 13 - "Community 13"
Cohesion: 0.06
Nodes (24): VehicleGPS2FactGroup, VehicleGPS2FactGroup::_handleGps2Raw(), VehicleGPS2FactGroup::handleMessage(), public, VehicleGPSFactGroup, _authenticationStateFact, _correctionsQualityFact, _countFact (+16 more)

### Community 15 - "Community 15"
Cohesion: 0.05
Nodes (19): QThread, UDPConfiguration::UDPConfiguration(), UDPLink, _disconnectedEmitted, public, _udpConfig, UDPLink::UDPLink(), _worker (+11 more)

### Community 16 - "Community 16"
Cohesion: 0.08
Nodes (23): Bootloader, _boardFlashSize, _boardID, boardIDPX4FMUV2, boardIDPX4FMUV3, boardIDSiKRadio1000, boardIDSiKRadio1060, Bootloader::Bootloader() (+15 more)

### Community 17 - "Community 17"
Cohesion: 0.06
Nodes (18): LinkInfo_t, commLost, heartbeatElapsedTimer, link, Vehicle, VehicleLinkManager, _allLinksRemovedSignalledByCloseVehicle, _autoDisconnect (+10 more)

### Community 18 - "Community 18"
Cohesion: 0.04
Nodes (25): MAVLinkLogManager, _currentLogfile, kDefaultDescr, kDefaultPx4URL, kDescriptionsKey, kEmailAddressKey, kEnableAutoStartKey, kEnableAutoUploadKey (+17 more)

### Community 19 - "Community 19"
Cohesion: 0.08
Nodes (18): function, LambdaFallbackHandlerData, showError, unsupportedLambda, vehicle, lambdaFallbackResultHandler(), MavCommandQueue, _ackTimeoutMSecsHighLatency (+10 more)

### Community 20 - "Community 20"
Cohesion: 0.05
Nodes (21): Actuator, public, ActuatorState, lastUpdated, state, value, ActuatorTest, _active (+13 more)

### Community 21 - "Community 21"
Cohesion: 0.05
Nodes (10): Actuators, _initError, _jsonMetadata, _motorAssignment, public, _subscribedFacts, _usedMixerLabels, _vehicle (+2 more)

### Community 22 - "Community 22"
Cohesion: 0.06
Nodes (14): ActuatorOutput, _groupVisibilityCondition, _label, _notes, _params, public, ActuatorOutputChannel, _label (+6 more)

### Community 23 - "Community 23"
Cohesion: 0.04
Nodes (23): BluetoothConfiguration, _adapterEnumerationInProgress, _adapterWatcher, _availableAdapters, _deferredPowerOnFixPending, _device, _deviceDiscoveryAgent, _deviceList (+15 more)

### Community 24 - "Community 24"
Cohesion: 0.06
Nodes (16): QThread, QTimer, SerialConfiguration::SerialConfiguration(), SerialLink, _disconnectedEmitted, public, _serialConfig, SerialLink::SerialLink() (+8 more)

### Community 25 - "Community 25"
Cohesion: 0.07
Nodes (19): AudioOutput, AudioOutputTest, _engine, _initialized, kEngineInitWarnTimeout, kMaxTextQueueSize, _lastVolume, _mutedFact (+11 more)

### Community 26 - "Community 26"
Cohesion: 0.05
Nodes (23): Fact, MotorAssignment, _actuators, _assignMotors, _commandInProgress, _firstMotorsFunction, _functionFacts, _message (+15 more)

### Community 27 - "Community 27"
Cohesion: 0.07
Nodes (22): VehicleEstimatorStatusFactGroup, _accelErrorFact, _goodAttitudeEstimateFact, _goodConstPosModeEstimateFact, _goodHorizPosAbsEstimateFact, _goodHorizPosRelEstimateFact, _goodHorizVelEstimateFact, _goodPredHorizPosAbsEstimateFact (+14 more)

### Community 28 - "Community 28"
Cohesion: 0.08
Nodes (14): InitialConnectStateMachine, _autopilotVersionMaxRetries, InitialConnectStateMachine::InitialConnectStateMachine(), _lastSkipReason, public, _stateAutopilotVersion, _stateCompInfo, _stateComplete (+6 more)

### Community 29 - "Community 29"
Cohesion: 0.05
Nodes (17): Bootloader, PX4FirmwareUpgradeThreadController::PX4FirmwareUpgradeThreadController(), PX4FirmwareUpgradeThreadWorker, _bootloader, _controller, _elapsed, _findBoardFirstAttempt, _findBoardTimer (+9 more)

### Community 30 - "Community 30"
Cohesion: 0.08
Nodes (21): VehicleEFIFactGroup, _baroPressFact, _cylinderTempFact, _ecuIndexFact, _engineLoadFact, _exGasTempFact, _fuelConsumedFact, _fuelFlowFact (+13 more)

### Community 31 - "Community 31"
Cohesion: 0.12
Nodes (5): ParsedEvent, EventHandler, ParameterManager, VehicleTypes, versionNotSetValue

### Community 33 - "Community 33"
Cohesion: 0.09
Nodes (11): SigningController, Vehicle, VehicleSigningController, _active, kRetransmitIntervalMs, _pendingHasKey, _pendingKey, QML_ELEMENT (+3 more)

### Community 35 - "Community 35"
Cohesion: 0.06
Nodes (19): MavCommandQueue, MessageIntervalManager, _commandQueue, _mavlinkMsgIntervals, MessageIntervalManager::MessageIntervalManager(), public, _reqMsgCoord, _unsupportedMessageIds (+11 more)

### Community 36 - "Community 36"
Cohesion: 0.06
Nodes (23): Action, Action::Action(), _commandInProgress, _label, _outputFunction, public, _type, _vehicle (+15 more)

### Community 37 - "Community 37"
Cohesion: 0.05
Nodes (33): AsyncFunctionState, CompInfo, ComponentInformationManager, ConditionalState, QGCState, RequestMetaDataTypeStateMachine, _activeAsyncState, _activeSkippableState (+25 more)

### Community 38 - "Community 38"
Cohesion: 0.09
Nodes (14): FirmwareImage, _binFormat, _boardId, FirmwareImage::FirmwareImage(), _ihxBlocks, _jsonAirframeXmlKey, _jsonAirframeXmlSizeKey, _jsonBoardIdKey (+6 more)

### Community 40 - "Community 40"
Cohesion: 0.07
Nodes (9): LogReplayLink, LogReplayLinkController, kNoPlayhead, LogReplayLinkController::LogReplayLinkController(), _playbackSpeed, _playheadSecs, _playheadTime, QML_ELEMENT (+1 more)

### Community 41 - "Community 41"
Cohesion: 0.09
Nodes (24): callbackForPolicy(), _channelSigningPtr(), checkSigningLinkId(), _computeSignatureHash(), createSetupSigning(), currentSigningTimestampTicks(), encodeSetupSigning(), insecureConnectionAcceptUnsignedCallback() (+16 more)

### Community 42 - "Community 42"
Cohesion: 0.08
Nodes (21): LinkManager::_addSerialAutoConnectLink(), LinkManager::_updateSerialPorts(), QGCSerialPortInfo, _boardDescriptionFallbackList, _boardInfoList, _boardManufacturerFallbackList, _jsonAndroidOnlyKey, _jsonBoardClassKey (+13 more)

### Community 43 - "Community 43"
Cohesion: 0.10
Nodes (8): AutoConnectSettings, containsHost(), containsTarget(), UDPClient, address, hostname, port, UDPConfiguration

### Community 45 - "Community 45"
Cohesion: 0.06
Nodes (28): ArduCopterFirmwarePlugin, _acroFlightMode, _altHoldFlightMode, _autoFlightMode, _autoRotateFlightMode, _autoRTLFlightMode, _autotuneFlightMode, _avoidADSBFlightMode (+20 more)

### Community 46 - "Community 46"
Cohesion: 0.09
Nodes (15): VehicleGeneratorFactGroup, _batCurrentSetpointFact, _batteryCurrentFact, _busVoltageFact, _flagsListGenerator, _genSpeedFact, _genTempFact, _loadCurrentFact (+7 more)

### Community 47 - "Community 47"
Cohesion: 0.10
Nodes (8): ParameterMetaData, _cachedMetaData, kEmptyDefines, ParameterMetaData::ParameterMetaData(), _parameterMetaDataLoaded, ValueDescPair, description, value

### Community 48 - "Community 48"
Cohesion: 0.05
Nodes (37): VehicleFactGroup, _airSpeedFact, _airSpeedSetpointFact, _altitudeAboveTerrFact, _altitudeAMSLFact, _altitudeMessageAvailable, _altitudeRelativeFact, _altitudeTuningFact (+29 more)

### Community 49 - "Community 49"
Cohesion: 0.06
Nodes (14): SigningController, _autoDetectGuard, _badSigBurst, _channel, _fsmMutex, kBadSignatureAlertThreshold, kTimeout, kWallClockRefreshInterval (+6 more)

### Community 50 - "Community 50"
Cohesion: 0.06
Nodes (9): Vehicle, VehicleObjectAvoidance, VehicleObjectAvoidance::grid(), kColPrevParam, _objDistance, _objGrid, QML_ELEMENT, _vehicle (+1 more)

### Community 51 - "Community 51"
Cohesion: 0.07
Nodes (28): AsyncFunctionState, CompInfo, CompInfoGeneral, CompInfoParam, ComponentInformationCache, ComponentInformationManager, _cachedFileDownload, cachedFileMaxAgeSec (+20 more)

### Community 52 - "Community 52"
Cohesion: 0.10
Nodes (6): BluetoothClassicWorker, _classicDiscoveredService, _classicDiscovery, public, _socket, SPP_UUID

### Community 53 - "Community 53"
Cohesion: 0.08
Nodes (19): ChannelConfig, _parameter, public, _visibilityCondition, ChannelConfigInstance, public, ConfigParameter, _label (+11 more)

### Community 55 - "Community 55"
Cohesion: 0.11
Nodes (6): LinkManager, QThread, SigningController, Vehicle, SigningController, VehicleLinkManagerTest

### Community 56 - "Community 56"
Cohesion: 0.09
Nodes (10): BluetoothLink, _bluetoothConfig, BluetoothLink::BluetoothLink(), _connectedCache, _disconnectedEmitted, public, _worker, _workerThread (+2 more)

### Community 57 - "Community 57"
Cohesion: 0.06
Nodes (16): BluetoothWorker, _consecutiveFailures, _device, _intentionalDisconnect, MAX_CONSECUTIVE_FAILURES, MAX_RECONNECT_ATTEMPTS, MAX_RECONNECT_INTERVAL_MS, public (+8 more)

### Community 58 - "Community 58"
Cohesion: 0.10
Nodes (14): VehicleDistanceSensorFactGroup, _maxDistanceFact, _minDistanceFact, _rotationNoneFact, _rotationPitch270Fact, _rotationPitch90Fact, _rotationYaw135Fact, _rotationYaw180Fact (+6 more)

### Community 59 - "Community 59"
Cohesion: 0.07
Nodes (18): ArduSubFirmwarePlugin, _acroFlightMode, _altHoldFlightMode, _autoFlightMode, _circleFlightMode, _factRenameMap, _guidedFlightMode, _infoFactGroup (+10 more)

### Community 60 - "Community 60"
Cohesion: 0.06
Nodes (10): LinkInterface, _config, kMavlinkV1TrafficGraceMsecsDefault, _mavlinkV1FirstSeenTimer, _mavlinkV1TrafficGraceMsecs, _mavlinkV2TrafficSeen, QML_ELEMENT, _signingController (+2 more)

### Community 61 - "Community 61"
Cohesion: 0.06
Nodes (16): LogReplayWorker, kTimestamp, _logCurrentTimeUSecs, _logDurationUSecs, _logEndTimeUSecs, _logFile, _logFileSize, _logReplayConfig (+8 more)

### Community 62 - "Community 62"
Cohesion: 0.09
Nodes (10): generateTestGeometries(), ImagePosition, index, position, radius, type, VehicleGeometryImageProvider, _actuatorImagePositions (+2 more)

### Community 65 - "Community 65"
Cohesion: 0.09
Nodes (9): EventHandler, HealthAndArmingCheckReport, MAVLinkEventManager, _events, MAVLinkEventManager::MAVLinkEventManager(), public, _vehicle, ParsedEvent (+1 more)

### Community 66 - "Community 66"
Cohesion: 0.13
Nodes (3): MAVLinkLogFiles::MAVLinkLogFiles(), MAVLinkLogManager::MAVLinkLogManager(), MAVLinkLogProcessor::MAVLinkLogProcessor()

### Community 67 - "Community 67"
Cohesion: 0.06
Nodes (29): ArduPlaneFirmwarePlugin, _acroFlightMode, _autoFlightMode, _autolandFlightMode, _autoTuneFlightMode, _avoidADSBFlightMode, _circleFlightMode, _cruiseFlightMode (+21 more)

### Community 69 - "Community 69"
Cohesion: 0.11
Nodes (4): CompInfoActuators::CompInfoActuators(), CompInfoGeneral::CompInfoGeneral(), CompInfoParam::CompInfoParam(), QJsonDocument

### Community 70 - "Community 70"
Cohesion: 0.11
Nodes (3): BluetoothConfiguration::getAllAvailableAdapters(), BluetoothConfiguration::getAllPairedDevices(), BluetoothConfiguration::getConnectedDevices()

### Community 71 - "Community 71"
Cohesion: 0.07
Nodes (30): Mode, ACRO, ALT_HOLD, AUTO, AUTO_RTL, AUTOROTATE, AUTOTUNE, AVOID_ADSB (+22 more)

### Community 73 - "Community 73"
Cohesion: 0.08
Nodes (5): FirmwareImage, PX4FirmwareUpgradeThreadController, public, _worker, _workerThread

### Community 75 - "Community 75"
Cohesion: 0.11
Nodes (15): DesiredStreamRate, messageId, rate, MAVLinkStreamConfig, _changedIds, _desiredRates, MAVLinkStreamConfig::MAVLinkStreamConfig(), _messageIntervalCb (+7 more)

### Community 76 - "Community 76"
Cohesion: 0.07
Nodes (18): FTPManager, _ackOrNakTimeoutMsecs, _ackOrNakTimeoutTimer, _currentStateMachineIndex, _deleteState, _downloadState, _expectedIncomingSeqNumber, friend (+10 more)

### Community 77 - "Community 77"
Cohesion: 0.07
Nodes (11): RemoteIDManager, _enforceSendingSelfID, _gcsPositionError, _id_or_mac_unknown, _odidTimeoutTimer, QML_ELEMENT, _sendMessagesTimer, _settings (+3 more)

### Community 78 - "Community 78"
Cohesion: 0.07
Nodes (28): Mode, ACRO, AUTO, AUTOLAND, AUTOTUNE, AVOID_ADSB, CIRCLE, CRUISE (+20 more)

### Community 79 - "Community 79"
Cohesion: 0.11
Nodes (7): Autotune, Autotune::Autotune(), _disarmMessageDisplayed, _pollTimer, QML_ELEMENT, _vehicle, Vehicle

### Community 80 - "Community 80"
Cohesion: 0.08
Nodes (11): TrajectoryPoints, _azimuthTolerance, _distanceTolerance, _lastAzimuth, _lastPoint, _points, QML_ELEMENT, QVariantList (+3 more)

### Community 81 - "Community 81"
Cohesion: 0.07
Nodes (13): QTimer, StatusTextHandler, m_activeComponent, m_chunkedStatusTextInfoMap, m_chunkedStatusTextTimer, m_errorCount, m_errorCountTotal, m_messageCount (+5 more)

### Community 83 - "Community 83"
Cohesion: 0.12
Nodes (6): FirmwarePlugin, Vehicle, VehicleSupports, QML_ELEMENT, _vehicle, VehicleSupports::VehicleSupports()

### Community 85 - "Community 85"
Cohesion: 0.08
Nodes (15): MAVLinkProtocol, _firstMessageSeen, _initialized, kMaxCompId, _lastIndex, _logFileExtension, _logSuspendError, _logSuspendReplay (+7 more)

### Community 86 - "Community 86"
Cohesion: 0.08
Nodes (6): LogReplayLink, _disconnectedEmitted, _logReplayConfig, public, _worker, _workerThread

### Community 87 - "Community 87"
Cohesion: 0.18
Nodes (7): APMParameterMetaData, APMParameterMetaData::APMParameterMetaData(), _rawParams, RawParamData, fields, group, QJsonObject

### Community 89 - "Community 89"
Cohesion: 0.08
Nodes (9): HealthAndArmingCheckProblem, _description, _message, _severity, HealthAndArmingCheckReport, _gpsState, _missionModeGroup, _takeoffModeGroup (+1 more)

### Community 90 - "Community 90"
Cohesion: 0.10
Nodes (7): ChannelConfigInstance, public, _visibleAxis, ChannelConfigInstanceVirtualAxis, _axes, _ignoreChange, public

### Community 91 - "Community 91"
Cohesion: 0.12
Nodes (12): BatteryFactGroup, _batteryFunctionFact, _batteryTypeFact, _chargeStateFact, _currentFact, _instantPowerFact, _mahConsumedFact, _percentRemainingFact (+4 more)

### Community 92 - "Community 92"
Cohesion: 0.10
Nodes (10): MAVLinkLogProcessor, _error, _file, _gotHeader, kSequenceSize, kUlogMessageHeader, _numDrops, _sequence (+2 more)

### Community 93 - "Community 93"
Cohesion: 0.10
Nodes (10): LinkInterface, MultiVehicleManager, _gcsHeartbeatTimer, _ignoreVehicleIds, _initialized, kGCSHeartbeatRateMSecs, QML_ELEMENT, QmlObjectListModel (+2 more)

### Community 95 - "Community 95"
Cohesion: 0.08
Nodes (8): MavlinkFTP, burstComplete, offset, opcode, paddng, req_opcode, session, size

### Community 97 - "Community 97"
Cohesion: 0.11
Nodes (12): EventHandler::EventHandler(), _impl, compid, handleEventCB, healthAndArmingChecks, healthAndArmingChecksValid, parser, pendingEvents (+4 more)

### Community 98 - "Community 98"
Cohesion: 0.09
Nodes (11): APMCustomMode, APMFirmwarePlugin, _adjustOutgoingMavlinkMutex, _ardupilotComponentMap, _artooIP, _artooVideoHandshakePort, _autoFlightMode, _coaxialMotors (+3 more)

### Community 99 - "Community 99"
Cohesion: 0.17
Nodes (5): ComponentInformationTranslation, _cachedFileDownload, ComponentInformationTranslation::ComponentInformationTranslation(), _toTranslateJsonFile, QGCCachedFileDownload

### Community 100 - "Community 100"
Cohesion: 0.09
Nodes (17): ActuatorOutput::ActuatorOutput(), ActuatorOutputChannel::ActuatorOutputChannel(), Condition, _alwaysTrueReason, _label, _operation, _parameter, _value (+9 more)

### Community 101 - "Community 101"
Cohesion: 0.12
Nodes (9): FactBitset, _ignoreChange, _integerFact, _offset, public, FactFloatAsBool, _floatFact, _ignoreChange (+1 more)

### Community 102 - "Community 102"
Cohesion: 0.11
Nodes (5): ConfigParameter, public, MixerConfigGroup, _params, public

### Community 104 - "Community 104"
Cohesion: 0.14
Nodes (8): VehicleRPMFactGroup, _rpm1Fact, _rpm2Fact, _rpm3Fact, _rpm4Fact, _rpmSensor1Fact, _rpmSensor2Fact, VehicleRPMFactGroup::VehicleRPMFactGroup()

### Community 106 - "Community 106"
Cohesion: 0.09
Nodes (23): Mode, ACRO, ALT_HOLD, AUTO, CIRCLE, GUIDED, MANUAL, MOTORDETECTION (+15 more)

### Community 107 - "Community 107"
Cohesion: 0.13
Nodes (6): FirmwarePlugin, FirmwarePluginFactory, FirmwarePluginManager, FirmwarePluginManager::FirmwarePluginManager(), _genericFirmwarePlugin, public

### Community 108 - "Community 108"
Cohesion: 0.10
Nodes (15): StickFunction, stickFunctionAdditionalAxis1, stickFunctionAdditionalAxis2, stickFunctionAdditionalAxis3, stickFunctionAdditionalAxis4, stickFunctionAdditionalAxis5, stickFunctionAdditionalAxis6, stickFunctionMax (+7 more)

### Community 109 - "Community 109"
Cohesion: 0.10
Nodes (11): LinkManager::serialBaudRates(), LinkManager::serialPorts(), LinkManager::serialPortStrings(), LogReplayLink, MAVLinkProtocol, QmlObjectListModel, QTimer, SerialLink (+3 more)

### Community 110 - "Community 110"
Cohesion: 0.10
Nodes (13): QGCMAVLink, FirmwareClassArduPilot, FirmwareClassGeneric, FirmwareClassPX4, public, type, VehicleClassAirship, VehicleClassFixedWing (+5 more)

### Community 111 - "Community 111"
Cohesion: 0.14
Nodes (5): LogReplayConfiguration, LogReplayConfiguration::LogReplayConfiguration(), QML_ELEMENT, LogReplayLink::LogReplayLink(), LogReplayWorker::LogReplayWorker()

### Community 112 - "Community 112"
Cohesion: 0.13
Nodes (10): EscStatusFactGroup, _connectionTypeFact, _countFact, _currentFact, _errorCountFact, _failureFlagsFact, _infoFact, _rpmFact (+2 more)

### Community 113 - "Community 113"
Cohesion: 0.12
Nodes (8): VehicleLocalPositionFactGroup, VehicleLocalPositionFactGroup::VehicleLocalPositionFactGroup(), _vxFact, _vyFact, _vzFact, _xFact, _yFact, _zFact

### Community 114 - "Community 114"
Cohesion: 0.12
Nodes (8): VehicleLocalPositionSetpointFactGroup, VehicleLocalPositionSetpointFactGroup::VehicleLocalPositionSetpointFactGroup(), _vxFact, _vyFact, _vzFact, _xFact, _yFact, _zFact

### Community 115 - "Community 115"
Cohesion: 0.15
Nodes (8): VehicleVibrationFactGroup, _clipCount1Fact, _clipCount2Fact, _clipCount3Fact, VehicleVibrationFactGroup::VehicleVibrationFactGroup(), _xAxisFact, _yAxisFact, _zAxisFact

### Community 116 - "Community 116"
Cohesion: 0.09
Nodes (18): MavCommandListEntry, ackHandlerInfo, ackTimeoutMSecs, command, elapsedTimer, frame, maxTries, rgParam1 (+10 more)

### Community 117 - "Community 117"
Cohesion: 0.12
Nodes (5): ImageProtocolManager, _imageBytes, _imageHandshake, ImageProtocolManager::ImageProtocolManager(), public

### Community 119 - "Community 119"
Cohesion: 0.10
Nodes (19): LinkManager::_allowAutoConnectToBoard(), BoardClassString2BoardType_t, boardType, classString, BoardInfo_t, boardType, name, productId (+11 more)

### Community 120 - "Community 120"
Cohesion: 0.10
Nodes (18): ArduRoverFirmwarePlugin, _acroFlightMode, _autoFlightMode, _circleFlightMode, _dockFlightMode, _guidedFlightMode, _holdFlightMode, _initializingFlightMode (+10 more)

### Community 121 - "Community 121"
Cohesion: 0.11
Nodes (19): ActuatorGroup, actuatorType, count, fixedCount, groupLabel, itemLabelPrefix, parameters, perItemParameters (+11 more)

### Community 123 - "Community 123"
Cohesion: 0.20
Nodes (5): VehicleTemperatureFactGroup, _temperature1Fact, _temperature2Fact, _temperature3Fact, VehicleTemperatureFactGroup::VehicleTemperatureFactGroup()

### Community 124 - "Community 124"
Cohesion: 0.10
Nodes (10): SigningChannel, _autoDetectSuspended, _detectCooldown, _enabled, kDetectCooldownMs, kPersistedTimestampSafetyBumpTicks, _lastTransitionStatus, _lock (+2 more)

### Community 125 - "Community 125"
Cohesion: 0.10
Nodes (11): SigningStatus, enabled, keyName, state, statusText, streamCount, State, Disabling (+3 more)

### Community 126 - "Community 126"
Cohesion: 0.14
Nodes (4): Joystick, JoystickConfigController, JoystickConfigController::JoystickConfigController(), QML_ELEMENT

### Community 127 - "Community 127"
Cohesion: 0.11
Nodes (7): QThread, TCPLink, _disconnectedEmitted, public, _tcpConfig, _worker, _workerThread

### Community 128 - "Community 128"
Cohesion: 0.14
Nodes (10): APMSubmarineFactGroup, _camTiltFact, _inputHoldFact, _lightsLevel1Fact, _lightsLevel2Fact, _pilotGainFact, _rangefinderDistanceFact, _rangefinderTargetFact (+2 more)

### Community 130 - "Community 130"
Cohesion: 0.10
Nodes (16): RequestMessageInfo, commandAckReceived, compId, coordinator, message, messageReceived, messageWaitElapsedTimer, msgId (+8 more)

### Community 131 - "Community 131"
Cohesion: 0.14
Nodes (4): ChannelConfig, _instances, public, ChannelConfigVirtualAxis

### Community 132 - "Community 132"
Cohesion: 0.11
Nodes (17): ActuatorGeometry, index, labelIndexOffset, position, renderOptions, spinDirection, type, RenderOptions (+9 more)

### Community 133 - "Community 133"
Cohesion: 0.11
Nodes (6): QTcpSocket, TCPWorker, _config, _errorEmitted, public, _socket

### Community 134 - "Community 134"
Cohesion: 0.13
Nodes (7): CommandSupportedResult, SUPPORTED, UNKNOWN, UNSUPPORTED, FirmwarePluginInstanceData, MAV_CMD_supported, public

### Community 135 - "Community 135"
Cohesion: 0.11
Nodes (11): Mixers, _actuatorTypes, _functions, _functionsSpecificLabel, _mixerConditions, _mixerOptions, _parameterManager, _rules (+3 more)

### Community 136 - "Community 136"
Cohesion: 0.12
Nodes (12): CompInfoParam, _indexedNameMetaDataList, kIndexedNameTag, kJsonParametersKey, _nameToMetaDataMap, _noJsonMetadata, _parameterMetaData, public (+4 more)

### Community 137 - "Community 137"
Cohesion: 0.13
Nodes (10): VehicleGPSAggregateFactGroup, _authenticationStateFact, _connections, GNSS_INTEGRITY_STALE_TIMEOUT_MS, _gps1, _gps2, _isStaleFact, _jammingStateFact (+2 more)

### Community 138 - "Community 138"
Cohesion: 0.11
Nodes (15): DownloadState_t, bytesWritten, checksize, expectedOffset, file, fileName, fileSize, fullPathOnVehicle (+7 more)

### Community 139 - "Community 139"
Cohesion: 0.12
Nodes (13): ComponentInformationCache, _cachedFiles, _cacheExtension, _maxNumFiles, _metaExtension, _nextAccessCounter, _numFiles, _path (+5 more)

### Community 143 - "Community 143"
Cohesion: 0.13
Nodes (6): SensorInfo, enabled, healthy, SysStatusSensorInfo, _sensorInfoMap, SysStatusSensorInfo::SysStatusSensorInfo()

### Community 146 - "Community 146"
Cohesion: 0.18
Nodes (4): EscStatusFactGroup::EscStatusFactGroup(), EscStatusFactGroupListModel, EscStatusFactGroupListModel::EscStatusFactGroupListModel(), QML_ELEMENT

### Community 147 - "Community 147"
Cohesion: 0.17
Nodes (5): VehicleHygrometerFactGroup, _hygroHumiFact, _hygroIDFact, _hygroTempFact, VehicleHygrometerFactGroup::VehicleHygrometerFactGroup()

### Community 148 - "Community 148"
Cohesion: 0.12
Nodes (13): TerrainAtCoordinateQuery, TerrainQueryCoordinator, _altLastCoord, _altLastRelAlt, _altQuery, _altQueryTimer, _doSetHomeCoordinate, _doSetHomeQuery (+5 more)

### Community 150 - "Community 150"
Cohesion: 0.12
Nodes (17): StateMachineStepFunction, StateMachineStepComplete, StateMachineStepExtensionHighHorz, StateMachineStepExtensionHighVert, StateMachineStepExtensionLowHorz, StateMachineStepExtensionLowVert, StateMachineStepPitchCenter, StateMachineStepPitchDown (+9 more)

### Community 153 - "Community 153"
Cohesion: 0.17
Nodes (5): UdpIODevice, _buffer, public, UdpIODevice::UdpIODevice(), QUdpSocket

### Community 155 - "Community 155"
Cohesion: 0.12
Nodes (16): Mode, ACRO, AUTO, CIRCLE, DOCK, FOLLOW, GUIDED, HOLD (+8 more)

### Community 156 - "Community 156"
Cohesion: 0.12
Nodes (10): DeleteFileState_t, fullPathOnVehicle, retryCount, ListDirectoryState_t, expectedOffset, fullPathOnVehicle, opCode, retryCount (+2 more)

### Community 157 - "Community 157"
Cohesion: 0.17
Nodes (3): EventHandler, public, HealthAndArmingChecks

### Community 159 - "Community 159"
Cohesion: 0.20
Nodes (5): VehicleClockFactGroup, _currentDateFact, _currentTimeFact, _currentUTCTimeFact, VehicleClockFactGroup::VehicleClockFactGroup()

### Community 162 - "Community 162"
Cohesion: 0.13
Nodes (15): CalibrationType, CalibrationAccel, CalibrationAPMAccelSimple, CalibrationAPMCompassMot, CalibrationAPMPreFlight, CalibrationAPMPressureAirspeed, CalibrationCopyTrims, CalibrationEsc (+7 more)

### Community 165 - "Community 165"
Cohesion: 0.13
Nodes (10): DisplayOption, Bitset, BoolTrueIfPositive, Default, Parameter, advanced, displayOption, indexOffset (+2 more)

### Community 166 - "Community 166"
Cohesion: 0.14
Nodes (6): Fact, FirmwareImage, FirmwareUpgradeController::FirmwareUpgradeController(), PX4FirmwareUpgradeThread, PX4FirmwareUpgradeThreadController, QGCFileDownload

### Community 167 - "Community 167"
Cohesion: 0.15
Nodes (5): AutoPilotPlugin, Autotune, MavlinkCameraControlInterface, QGCCameraManager, VehicleComponent

### Community 168 - "Community 168"
Cohesion: 0.18
Nodes (10): QTimer, TerrainFactGroup, TerrainProtocolHandler, _currentTerrainRequest, public, _terrainDataSendTimer, _terrainFactGroup, _terrainRequestActive (+2 more)

### Community 169 - "Community 169"
Cohesion: 0.18
Nodes (3): TCPConfiguration::TCPConfiguration(), TCPLink::TCPLink(), TCPWorker::TCPWorker()

### Community 170 - "Community 170"
Cohesion: 0.21
Nodes (5): FirmwarePluginFactory, public, FirmwarePluginFactoryRegister, _factoryList, public

### Community 172 - "Community 172"
Cohesion: 0.14
Nodes (13): Rule, applyIdentifiers, items, selectIdentifier, RuleItem, defaultVal, disabled, hasDefault (+5 more)

### Community 173 - "Community 173"
Cohesion: 0.14
Nodes (5): CompInfo, compId, public, type, _uris

### Community 174 - "Community 174"
Cohesion: 0.15
Nodes (6): CompInfoEvents, CompInfoEvents::CompInfoEvents(), public, FactMetaData, FirmwarePlugin, Vehicle

### Community 178 - "Community 178"
Cohesion: 0.15
Nodes (6): FirmwareParameterHeader, firmwareType, gitRevision, vehicleType, versionNumber, versionType

### Community 179 - "Community 179"
Cohesion: 0.22
Nodes (10): APMFirmwarePluginFactory, _arduCopterPluginInstance, _arduPlanePluginInstance, _arduRoverPluginInstance, _arduSubPluginInstance, public, ArduCopterFirmwarePlugin, ArduPlaneFirmwarePlugin (+2 more)

### Community 180 - "Community 180"
Cohesion: 0.22
Nodes (5): StatusText, m_compId, m_formatedText, m_severity, m_text

### Community 182 - "Community 182"
Cohesion: 0.15
Nodes (8): MavCommandQueue, RequestMessageCoordinator, _commandQueue, _infoMap, public, _queueMap, _vehicle, Vehicle

### Community 184 - "Community 184"
Cohesion: 0.15
Nodes (8): StandardModes, _lastSeq, _modeList, public, _requestActive, _vehicle, _wantReset, Vehicle

### Community 185 - "Community 185"
Cohesion: 0.21
Nodes (3): BluetoothMode, ModeClassic, ModeLowEnergy

### Community 189 - "Community 189"
Cohesion: 0.17
Nodes (10): OpKind, Disable, Enable, None, PendingOp, expectedSysId, keyBytes, keyName (+2 more)

### Community 190 - "Community 190"
Cohesion: 0.17
Nodes (6): OutputFunction, actuatorType, excludeFromActuatorTesting, label, note, noteCondition

### Community 191 - "Community 191"
Cohesion: 0.17
Nodes (9): MixerChannel, _actuatorTypeIndex, _applyingRule, _currentSelectIdentifierValue, _label, _paramIndex, public, _rule (+1 more)

### Community 192 - "Community 192"
Cohesion: 0.17
Nodes (10): UploadState_t, cancelled, file, fileSize, fullPathOnVehicle, lastChunkSize, localFilePath, retryCount (+2 more)

### Community 194 - "Community 194"
Cohesion: 0.18
Nodes (5): FirmwareIdentifier, autopilotStackType, firmwareType, firmwareVehicleType, qHash()

### Community 195 - "Community 195"
Cohesion: 0.18
Nodes (3): QmlObjectListModel, QNetworkAccessManager, Vehicle

### Community 196 - "Community 196"
Cohesion: 0.18
Nodes (6): LinkType, TypeBluetooth, TypeLast, TypeLogReplay, TypeTcp, TypeUdp

### Community 200 - "Community 200"
Cohesion: 0.18
Nodes (7): Reason, InitFailed, Timeout, VehicleUnreachable, SigningFailure, detail, reason

### Community 201 - "Community 201"
Cohesion: 0.18
Nodes (11): ActuatorType, functionMax, functionMin, labelIndexOffset, perItemParams, values, Values, defaultVal (+3 more)

### Community 202 - "Community 202"
Cohesion: 0.20
Nodes (4): CompInfoGeneral, _jsonMetadataTypesKey, public, _supportedTypes

### Community 203 - "Community 203"
Cohesion: 0.22
Nodes (4): TerrainFactGroup, _blocksLoadedFact, _blocksPendingFact, TerrainFactGroup::TerrainFactGroup()

### Community 205 - "Community 205"
Cohesion: 0.20
Nodes (4): APMFirmwarePlugin::APMFirmwarePlugin(), ArduCopterFirmwarePlugin::ArduCopterFirmwarePlugin(), ArduPlaneFirmwarePlugin::ArduPlaneFirmwarePlugin(), ArduRoverFirmwarePlugin::ArduRoverFirmwarePlugin()

### Community 208 - "Community 208"
Cohesion: 0.20
Nodes (6): APMFirmwarePluginInstanceData, lastBatteryStatusTime, lastHomePositionTime, MAV_CMD_DO_REPOSITION_supported, MAV_CMD_DO_REPOSITION_unsupported, using

### Community 209 - "Community 209"
Cohesion: 0.20
Nodes (9): FirmwareCapabilities, ChangeHeadingCapability, GuidedModeCapability, GuidedTakeoffCapability, OrbitModeCapability, PauseVehicleCapability, ROIModeCapability, SetFlightModeCapability (+1 more)

### Community 210 - "Community 210"
Cohesion: 0.20
Nodes (9): Uris, crcMetaData, crcMetaDataFallback, crcMetaDataFallbackValid, crcMetaDataValid, uriMetaData, uriMetaDataFallback, uriTranslation (+1 more)

### Community 211 - "Community 211"
Cohesion: 0.22
Nodes (7): FirmwareToUrlElement_t, firmwareType, stackType, url, vehicleType, FirmwareVehicleType_t, FirmwareUpgradeController::vehicleTypeFromFirmwareSelectionIndex()

### Community 213 - "Community 213"
Cohesion: 0.22
Nodes (8): FirmwareFlightMode, advanced, canBeSet, custom_mode, fixedWing, mode_name, multiRotor, standard_mode

### Community 215 - "Community 215"
Cohesion: 0.22
Nodes (9): rcCalStates, rcCalStateBegin, rcCalStateCenterThrottle, rcCalStateChannelWait, rcCalStateDetectInversion, rcCalStateIdentify, rcCalStateMinMax, rcCalStateSave (+1 more)

### Community 216 - "Community 216"
Cohesion: 0.29
Nodes (3): FirmwarePlugin, FirmwarePluginFactory::FirmwarePluginFactory(), FirmwarePluginFactoryRegister::instance()

### Community 218 - "Community 218"
Cohesion: 0.25
Nodes (5): CompInfoActuators, public, FactMetaData, FirmwarePlugin, Vehicle

### Community 220 - "Community 220"
Cohesion: 0.29
Nodes (5): StateMachineEntry, channelInputFn, nextButtonFn, stepFunction, stickFunction

### Community 221 - "Community 221"
Cohesion: 0.29
Nodes (5): MavCmdAckHandlerInfo_s, progressHandler, progressHandlerData, resultHandler, resultHandlerData

### Community 225 - "Community 225"
Cohesion: 0.29
Nodes (5): AsyncFunctionState, RetryableRequestMessageState, RetryState, SkippableAsyncState, Vehicle

### Community 226 - "Community 226"
Cohesion: 0.29
Nodes (7): AuthState, AUTH_DISABLED, AUTH_ERROR, AUTH_INITIALIZING, AUTH_INVALID, AUTH_OK, AUTH_UNKNOWN

### Community 227 - "Community 227"
Cohesion: 0.29
Nodes (4): StateFunctions_t, ackNakFn, beginFn, timeoutFn

### Community 229 - "Community 229"
Cohesion: 0.29
Nodes (7): ChannelInfo, channelMax, channelMin, channelReversed, channelTrim, deadband, stickFunction

### Community 230 - "Community 230"
Cohesion: 0.33
Nodes (4): DetectSnapshot, autoDetectSuspended, inCooldown, keyHint

### Community 233 - "Community 233"
Cohesion: 0.33
Nodes (4): CheckList, CheckListFailed, CheckListNotSetup, CheckListPassed

### Community 234 - "Community 234"
Cohesion: 0.33
Nodes (5): PIDTuningTelemetryMode, ModeAltitudeAndAirspeed, ModeDisabled, ModeRateAndAttitude, ModeVelocityAndPosition

### Community 235 - "Community 235"
Cohesion: 0.33
Nodes (6): BothSticksDisplayPositions, leftStick, rightStick, StickDisplayPosition, horizontal, vertical

### Community 239 - "Community 239"
Cohesion: 0.40
Nodes (5): Mode, AUTO, GUIDED, RTL, SMART_RTL

### Community 240 - "Community 240"
Cohesion: 0.40
Nodes (3): TimestampSnapshot, keyName, timestamp

### Community 241 - "Community 241"
Cohesion: 0.40
Nodes (4): CompInfoGeneral, FactMetaData, FirmwarePlugin, Vehicle

### Community 243 - "Community 243"
Cohesion: 0.40
Nodes (5): MetadataSource, Cache, FTP, HTTP, None

### Community 258 - "Community 258"
Cohesion: 0.50
Nodes (4): __ChunkedStatusTextInfo, chunkId, rgMessageChunks, severity

### Community 259 - "Community 259"
Cohesion: 0.50
Nodes (3): FactMetaData, FirmwarePlugin, Vehicle

### Community 260 - "Community 260"
Cohesion: 0.50
Nodes (4): WithTimeSupport_t, Supported, Unknown, Unsupported

### Community 261 - "Community 261"
Cohesion: 0.50
Nodes (4): LocationTypes, FIXED, LiveGNSS, TAKEOFF

## Knowledge Gaps
- **1902 isolated node(s):** `public`, `_controller`, `_service`, `_readCharacteristic`, `_writeCharacteristic` (+1897 more)
  These have ≤1 connection - possible missing edges. (Counts symbols only; 3338 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)
- **97 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Vehicle` connect `Community 0` to `Community 2`, `Community 134`, `Community 7`, `Community 149`, `Community 158`, `Community 31`, `Community 32`, `Community 33`, `Community 163`, `Community 39`, `Community 44`, `Community 48`, `Community 64`, `Community 69`, `Community 88`, `Community 222`, `Community 232`, `Community 233`, `Community 234`, `Community 109`?**
  _High betweenness centrality (0.095) - this node is a cross-community bridge._
- **Why does `FactGroup` connect `Community 2` to `Community 128`, `Community 129`, `Community 134`, `Community 137`, `Community 10`, `Community 13`, `Community 147`, `Community 27`, `Community 30`, `Community 159`, `Community 167`, `Community 46`, `Community 48`, `Community 58`, `Community 59`, `Community 188`, `Community 203`, `Community 104`, `Community 113`, `Community 114`, `Community 115`, `Community 118`, `Community 249`, `Community 123`?**
  _High betweenness centrality (0.079) - this node is a cross-community bridge._
- **Why does `QFile` connect `Community 158` to `Community 0`, `Community 6`, `Community 7`, `Community 138`, `Community 16`, `Community 152`, `Community 31`, `Community 38`, `Community 42`, `Community 55`, `Community 61`, `Community 63`, `Community 192`, `Community 66`, `Community 195`, `Community 68`, `Community 85`, `Community 92`, `Community 96`, `Community 99`, `Community 248`?**
  _High betweenness centrality (0.056) - this node is a cross-community bridge._
- **What connects `public`, `_controller`, `_service` to the rest of the system?**
  _1902 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Community 0` be split into smaller, more focused modules?**
  _Cohesion score 0.008874340789234407 - nodes in this community are weakly interconnected._
- **Should `Community 1` be split into smaller, more focused modules?**
  _Cohesion score 0.011049723756906077 - nodes in this community are weakly interconnected._
- **Should `Community 2` be split into smaller, more focused modules?**
  _Cohesion score 0.018246869409660107 - nodes in this community are weakly interconnected._