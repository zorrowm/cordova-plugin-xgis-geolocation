# cordova-plugin-xgis-geolocation

> **Android Only ! ! !**
>
> 基于cordova-plugin-advanced-geolocation改造，使用`GnssStatus`接口获取卫星数据，因为`GpsStatus` 和 `GpsSatellite` 在 Android 8.0 之后已被废弃。该插件只支持**Android**，支持Cordova和Capacitor调用。

## cordova-plugin-advanced-geolocation (Android-only)

Highly configurable native interface to [GPS](http://developer.android.com/reference/android/location/LocationManager.html#GPS_PROVIDER) and [NETWORK](http://developer.android.com/reference/android/location/LocationManager.html#NETWORK_PROVIDER) on-device [location providers](http://developer.android.com/reference/android/location/LocationProvider.html). It will return and identify any location data registered by the on-device providers including real-time [satellite info](http://developer.android.com/reference/android/location/GpsSatellite.html).

It also offers direct access to [CellInfo](http://developer.android.com/reference/android/telephony/CellInfo.html), [CellLocation](http://developer.android.com/reference/android/telephony/CellLocation.html) and [CellSignalStrength](https://developer.android.com/reference/android/telephony/CellSignalStrength.html) data.

In comparison to the W3C HTML Geolocation API, this plugin provides you with significantly greater control and more information to make better decisions with geolocation data.

If you are experiencing annoying jumping around of your geolocation coordinates then this plugin will help. It seperates GPS from NETWORK coordinates and allows you smooth out the user experience.

**API说明参考：**

https://www.npmjs.com/package/cordova-plugin-advanced-geolocation

https://github.com/zorrowm/cordova-plugin-xgis-geolocation/blob/HEAD/api_reference.md

## 基础模型类型

```ts
export interface IGPSPosition {
  /**
   * 位置来源,gps或network
   */
  provider: string;
  /**
   * 经度,单位：度
   */
  longitude: number;
  /**
   * 纬度，单位：度
   */
  latitude: number;
  /**
   * 高度，单位：米
   */
  altitude: number;
  /**
   * 精度，单位：米
   * 多少米
   */
  accuracy: number;

  /**
   * 方位角，单位：度
   */
  heading: number;
  /**
   * 速度，单位：m/s
   */
  speed: number;
  /**
   * 时间便签
   */
  timestamp: number;
  /**
   * 是否来自于缓存
   */
  cached: boolean;
}

/**
 * 卫星数据
 */
export interface ISatelliteData {

  /**
   * Android 中通过 GnssStatus.getSvid() 获取的是 SVID（Satellite Vehicle ID），并不是 PRN。
   */
  vid: number;
  /**
   * 卫星类型： 1：GPS，2：SBAS  3：GLONASS  4：QZSS  5：Beidou北斗  6：Galileo 伽利略  0:其他未知
   */
  satType: number;
  /**
   * 重启 GPS 引擎以来接收第一个定位所需的时间。
   */
  //   timeToFirstFix: number;
  /**
   * 是否使用卫星来确定 GPS 定位
   */
  usedInFix: boolean;
  /**
   * 卫星的方位角，单位为度。
   * 方位角可以在0到360之间变化。
   */
  azimuth: number;
  /**
   * 卫星在地平线上的海拔高度，单位为度。
   * 海拔高度可以在0到90之间变化。
   */
  elevation: number;
  /**
   * 卫星是否具有星历数据
   */
  hasEphemeris:boolean;
  /**
   * 星是否具有年历数据
   */
  hasAlmanac:boolean;
  /**
   * 卫星的信噪比
   */
  SNR: number;
}

export interface IGroupSatellites {
  /**
   * 时间，毫秒数，从1970.1.1起算
   */
  // timestamp:number,
  /**
   * 解析为日期
   */
  dateTime: string;
  /**
   * 卫星数
   */
  count: number;
  /**
   * 可用有效的卫星数
   */
  usedInFix:number;
  /**
   * 卫星列表
   */
  satellites: Array<ISatelliteData>;
}
```

## 插件调用示例代码（Capacitor）

- 包装plugin代码

```ts
import AdvancedGeolocation from 'cordova-plugin-xgis-geolocation/www/AdvancedGeolocation';
import {Global} from 'xframelib';
const emitter = Global.EventBus;
//参看说明：https://www.npmjs.com/package/cordova-plugin-advanced-geolocation?activeTab=readme
class CordovaAdvancedLocationPlugin {
  //GPS定位事件
  private static readonly GPS_LOCATION_EVENT = 'GPS_LOCATION_EVENT';
  //卫星事件
  private static readonly GPS_SATELLITE_EVENT = 'GPS_Satellite_EVENT';
  //定位出错事件
  private static readonly LOCATION_ERROR_EVENT = 'LOCATION_ERROR_EVENT';
  /**
   * 三个列表
   */
  private static GpsHandlerList:Set<any>=new Set();
//   private static SatelliteHandlerList:Set<any>=new Set();
//   private static ErrorHandlerList:Set<any>=new Set();
  private static _usedInFix:number=0;

  /**
   * 监听GPS位置变化
   * @param gpsLocationHandler
   */
  static onLocationHandler(gpsLocationHandler: (data: IGPSPosition) => void) {
    emitter.on(this.GPS_LOCATION_EVENT, gpsLocationHandler as any);
    this.GpsHandlerList.add(gpsLocationHandler);
  }
  static offLocationHandler(gpsLocationHandler: any) {
    emitter.off(this.GPS_LOCATION_EVENT, gpsLocationHandler);
    this.GpsHandlerList.delete(gpsLocationHandler);
  }

  /**
   * 监听GPS 卫星数据变化
   * @param gpsLocationHandler
   */
  static onSatelliteHandler(satelliteHandler: (data: IGroupSatellites) => void) {
    emitter.on(this.GPS_SATELLITE_EVENT, satelliteHandler as any);
  }
  static offSatelliteHandler(satelliteHandler: any) {
    emitter.off(this.GPS_SATELLITE_EVENT, satelliteHandler);

  }

  /**
   * 定位出错时的处理事件
   * @param locationErrorHandler
   */
  static onErrorHandler(locationErrorHandler:any) {
    emitter.on(this.LOCATION_ERROR_EVENT, locationErrorHandler);
  }
  static offErrorHandler(locationErrorHandler:any) {
    emitter.off(this.LOCATION_ERROR_EVENT, locationErrorHandler);
  }

  /**
   * 可用有效卫星数（参与定位的）
   */
  static get UsedSatellites():number
  {
    return this._usedInFix;
  }

  static start(provider:string="gps") {
    const that=this;
    AdvancedGeolocation.start(
      function (data:any) {
        try{
          var jsonObject = JSON.parse(data);
          switch (jsonObject.provider) {
            case 'network':
            case 'gps':
              if (jsonObject.latitude != '0.0') {
                const location: IGPSPosition = {
                  provider: jsonObject.provider,
                  longitude: Number(jsonObject.longitude),
                  latitude: Number(jsonObject.latitude),
                  altitude: Number(jsonObject.altitude),
                  accuracy: Number(jsonObject.accuracy),
                  heading: Number(jsonObject.bearing),
                  speed: Number(jsonObject.speed),
                  timestamp: Number(jsonObject.timestamp),
                  cached: Boolean(jsonObject.cached)
                };
                emitter.emit(that.GPS_LOCATION_EVENT, location);
              }
              break;
            case 'satellite':
            //可用卫星数
              that._usedInFix=Number(jsonObject["usedInFix"]);
              const count =Number(jsonObject["count"]);

              const date = new Date(parseInt(jsonObject['timestamp']));
              const dateTime = date.toUTCString();

              const satelliteList = new Array<ISatelliteData>();
              const groupSatelliteData: IGroupSatellites = {
                dateTime,
                count,
                usedInFix:that._usedInFix,
                satellites: satelliteList
              };
              const json = jsonObject;
              for(let i=0;i<count;i++)
              {
                const key=i.toString();

                const item = json[key];
                const satelliteData: ISatelliteData = {
                  vid:  Number(item.vid),
                  satType:  Number(item.satType),
                  usedInFix: Boolean(item.usedInFix),
                  azimuth: Number(item.azimuth),
                  elevation: Number(item.elevation),
                  hasEphemeris:Boolean(item.hasEphemeris),
                  hasAlmanac:Boolean(item.hasEphemeris),
                  SNR: Number(item.SNR)
                };
                satelliteList.push(satelliteData);
              }
              if (count> 0) {
                //发送卫星数据
                emitter.emit(that.GPS_SATELLITE_EVENT, groupSatelliteData);
              }
              break;
          }
        } catch (exc) {
          console.warn('Invalid JSON: ' + exc);
        }
      },
      function (error:any) {
        console.log('错误：',error)
        emitter.emit(that.LOCATION_ERROR_EVENT, error);
      },
      {
        minTime: 0,
        minDistance: 0,
        noWarn: true,
        providers: provider, //"some" Activates only GPS and Network providers.
        useCache: false,
        satelliteData: true,
        buffer: false,
        bufferSize: 0,
        signalStrength:false
      }
    );
  }

  //参看：https://github.com/esri/cordova-plugin-advanced-geolocation/blob/HEAD/api_reference.md
  static stop() {
    //返回结果如下
    // {
    //     "stopLocation":[
    //         {"provider":"gps","success":true},
    //         {"provider":"network","success":true},
    //         {"provider":"cell","success":true}
    //     ]
    // }
    return new Promise((resolve, reject) => {
      AdvancedGeolocation.stop(resolve, reject);
    });
  }

  static kill() {
    return new Promise((resolve, reject) => {
      AdvancedGeolocation.kill(resolve, reject);
    });
  }
}

export default CordovaAdvancedLocationPlugin;

```



- 调用示例代码

  ```vue
  <template>
      <WidgetBackPanel :isBack="false" class="markPointManager">
      <template #title>
        <div class="leftTitle">测试GPS获取数据</div>
      </template>
      <template #content>
  
      </template>
    </WidgetBackPanel>
  
  </template>
  
  <script lang="ts" setup>
  import WidgetBackPanel from 'src/components/WidgetBackPanel/index.vue';
  import CordovaAdvancedLocationPlugin, { IGPSPosition, IGroupSatellites } from 'src/plugin/CordovaAdvancedLocationPlugin';
  import { onMounted, onUnmounted } from 'vue';
  
  
  function locate(data:IGPSPosition)
  {
    console.log('位置数据为：',data);
  }
  function satelliteHandler( groupSatellites:IGroupSatellites)
  {
    console.log('卫星强度为：',CordovaAdvancedLocationPlugin.UsedSatellites);
    // console.log('卫星数据为：',groupSatellites);
  }
  
  onMounted( () => {
    CordovaAdvancedLocationPlugin.onLocationHandler(locate);
    CordovaAdvancedLocationPlugin.onSatelliteHandler(satelliteHandler);
    CordovaAdvancedLocationPlugin.start();
  })
  onUnmounted(() => {
    CordovaAdvancedLocationPlugin.offLocationHandler(locate);
    CordovaAdvancedLocationPlugin.offSatelliteHandler(satelliteHandler);
      //卸载时处理
      CordovaAdvancedLocationPlugin.stop();
  })
  </script>
  <style lang="scss" scoped>
  </style>
  ```

  



## 帮助资料

### GnssStatus接口开发

https://www.apiref.com/android-zh/android/location/GnssStatus.html

https://blog.csdn.net/2301_77563065/article/details/132733252

### SVID与PRN的映射关系：

- Android 中通过 `GnssStatus.getSvid()` 获取的是 **SVID**（Satellite Vehicle ID），并不是 PRN。

Android 提供了每颗卫星的类型信息，可以使用 GnssStatus.getConstellationType() 获取卫星所属系统。通过判断系统类型，再加上偏移量，即可实现映射。

下面是推荐的映射关系：

**卫星系统	ConstellationType 常量	         SVID 范围	映射到 PRN 的方法**
GPS	GnssStatus.CONSTELLATION_GPS	1~32	     直接使用，无需转换
北斗 BDS	CONSTELLATION_BEIDOU	     1~37	      SVID + 200 → PRN 201~237
GLONASS	CONSTELLATION_GLONASS	1~24	       SVID + 64 → PRN 65~88
Galileo	CONSTELLATION_GALILEO	       1~36	       可使用 SVID + 300（可选）


原文链接：https://blog.csdn.net/nysin/article/details/148846819
