title: ONVIF 客户端生成代码框架和简单封装
author: Cyrus
tags: []
categories: []
date: 2020-01-15 22:20:00
---
### 一、生成代码框架
ONVIF编译和简单代码，可以看看这个系列：https://blog.csdn.net/benkaoya/article/details/72424335 感谢博主的无私奉献，当然，如果源码下载积分可以低一点就更好了（没积分下载不了）。看这篇文章前可以先看看，方便理解。

练手，了解一下ONVIF,主要操作有，查找设备、获取设备信息、设备能力、设备profile和简单的ptz控制。因为上面的系列教程没有涉及ptz，所以在wsdl2h 生成头文件时，加上 http://www.onvif.org/onvif/ver20/ptz/wsdl/ptz.wsdl 以获取ptz相关功能。

### 二、简单封装
#### 1、创建一个device结构体，用来保存设备数据及相关方法：
OnvifDevice.h
~~~
#ifndef OnvifDevice_h
#define OnvifDevice_h

#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include "soapH.h"
#include "wsaapi.h"
#include "wsseapi.h"

#ifdef __cplusplus
extern "C"{
#endif

typedef struct oc_device {
    char *xAddr;		// 搜索得到的地址
    char *userName;		// 用户名
    char *password;		// 地址
    struct _tds__GetDeviceInformationResponse  devinfo;	// 设备信息
    struct tt__Capabilities  capabilities;		// 设备能力
    struct _trt__GetProfilesResponse profiles;	// profiles
}oc_device_t;

// 创建设备
oc_device_t *createDevice(const char *xAddr, const char *userName, const char *password);
// 删除设备
void destroyDevice(oc_device_t *device);

// 保存设备信息
void setInfo(oc_device_t *device, struct _tds__GetDeviceInformationResponse  *info);
// 删除设备信息
void removeInfo(oc_device_t *device);

// 保存设备能力
void setCapabilities(oc_device_t *device, struct tt__Capabilities  *capa);
// 删除设备能力
void removeCapabilities(oc_device_t *device);

// 保存设备Profile
void setProfiles(oc_device_t *device, struct _trt__GetProfilesResponse *profiles);
// 删除设备Profile
void removeProfiles(oc_device_t *device);
    
#ifdef __cplusplus
}
#endif
    
#endif /* OnvifDevice_h */

// 注意，设备相关操作需使用函数，不要直接操作结构体，否则会造成内存泄露
~~~

OnvifDevice.c
~~~
#include "OnvifDevice.h"

oc_device_t *createDevice(const char *xAddr, const char *userName, const char *password)
{
    assert(NULL != xAddr);
    assert(NULL != userName);
    assert(NULL != password);
    
    oc_device_t *device = malloc(sizeof(oc_device_t));
    memset(device, 0, sizeof(oc_device_t));
    
    device->xAddr = malloc(strlen(xAddr));
    bzero(device->xAddr, strlen(xAddr)+1);
    memcpy(device->xAddr, xAddr, strlen(xAddr));
    
    device->userName = malloc(strlen(userName));
    bzero(device->userName, strlen(userName)+1);
    memcpy(device->userName, userName, strlen(userName));
    
    device->password = malloc(strlen(password));
    bzero(device->password, strlen(password)+1);
    memcpy(device->password, password, strlen(password));
    return device;
}

void destroyDevice(oc_device_t *device)
{
    if (device == NULL) {
        return;
    }
    if (device->xAddr != NULL) {
        free(device->xAddr);
        device->xAddr = NULL;
    }
    if (device->userName != NULL) {
        free(device->userName);
        device->userName = NULL;
    }
    if (device->password != NULL) {
        free(device->password);
        device->password = NULL;
    }
    
    removeInfo(device);
    removeProfiles(device);
    removeCapabilities(device);
    
    free(device);
    device = NULL;
}

void setInfo(oc_device_t *device, struct _tds__GetDeviceInformationResponse  *info)
{
    if (device == NULL || info == NULL) {
        return;
    }
    
    removeInfo(device);
    if (info->Manufacturer != NULL) {
        device->devinfo.Manufacturer = malloc(strlen(info->Manufacturer)+1);
        bzero(device->devinfo.Manufacturer, strlen(info->Manufacturer)+1);
        memcpy(device->devinfo.Manufacturer, info->Manufacturer, strlen(info->Manufacturer));
    }
    
    
    if (info->FirmwareVersion != NULL) {
        device->devinfo.FirmwareVersion = malloc(strlen(info->FirmwareVersion)+1);
        bzero(device->devinfo.FirmwareVersion, strlen(info->FirmwareVersion)+1);
        memcpy(device->devinfo.FirmwareVersion, info->FirmwareVersion, strlen(info->FirmwareVersion));
    }
    
    
    if (info->SerialNumber != NULL) {
        device->devinfo.SerialNumber = malloc(strlen(info->SerialNumber)+1);
        bzero(device->devinfo.SerialNumber, strlen(info->SerialNumber)+1);
        memcpy(device->devinfo.SerialNumber, info->SerialNumber, strlen(info->SerialNumber));
    }
    
    
    if (info->HardwareId != NULL) {
        device->devinfo.HardwareId = malloc(strlen(info->HardwareId)+1);
        bzero(device->devinfo.HardwareId, strlen(info->HardwareId)+1);
        memcpy(device->devinfo.HardwareId, info->HardwareId, strlen(info->HardwareId));
    }
    
    
    if (info->HardwareId != NULL) {
        device->devinfo.Model = malloc(strlen(info->Model)+1);
        bzero(device->devinfo.Model, strlen(info->Model)+1);
        memcpy(device->devinfo.Model, info->Model, strlen(info->Model));
    }
}

void removeInfo(oc_device_t *device)
{
    if (device->devinfo.FirmwareVersion !=  NULL) {
        free(device->devinfo.FirmwareVersion);
        device->devinfo.FirmwareVersion = NULL;
    }
    if (device->devinfo.Manufacturer !=  NULL) {
        free(device->devinfo.Manufacturer);
        device->devinfo.Manufacturer = NULL;
    }
    if (device->devinfo.Model !=  NULL) {
        free(device->devinfo.Model);
        device->devinfo.Model = NULL;
    }
    if (device->devinfo.SerialNumber !=  NULL) {
        free(device->devinfo.SerialNumber);
        device->devinfo.SerialNumber = NULL;
    }
    if (device->devinfo.HardwareId !=  NULL) {
        free(device->devinfo.HardwareId);
        device->devinfo.HardwareId = NULL;
    }
    memset(&device->devinfo, 0, sizeof(device->devinfo));
}



void setCapabilities(oc_device_t *device, struct tt__Capabilities  *capa)
{
    if (device == NULL || capa == NULL) {
        return;
    }
    
    removeCapabilities(device);
    
    if (capa->Analytics->XAddr != NULL) {
        device->capabilities.Analytics = malloc(sizeof(struct tt__AnalyticsCapabilities));
        device->capabilities.Analytics->XAddr = malloc(strlen(capa->Analytics->XAddr)+1);
        bzero(device->capabilities.Analytics->XAddr, strlen(capa->Analytics->XAddr)+1);
        memcpy(device->capabilities.Analytics->XAddr, capa->Analytics->XAddr, strlen(capa->Analytics->XAddr));
    }
    
    if (capa->Device->XAddr != NULL) {
        device->capabilities.Device = malloc(sizeof(struct tt__DeviceCapabilities));
        device->capabilities.Device->XAddr = malloc(strlen(capa->Device->XAddr)+1);
        bzero(device->capabilities.Device->XAddr, strlen(capa->Device->XAddr)+1);
        memcpy(device->capabilities.Device->XAddr, capa->Device->XAddr, strlen(capa->Device->XAddr));
    }
    
    if (capa->Events->XAddr != NULL) {
        device->capabilities.Events = malloc(sizeof(struct tt__EventCapabilities));
        device->capabilities.Events->XAddr = malloc(strlen(capa->Events->XAddr)+1);
        bzero(device->capabilities.Events->XAddr, strlen(capa->Events->XAddr)+1);
        memcpy(device->capabilities.Events->XAddr, capa->Events->XAddr, strlen(capa->Events->XAddr));
    }
   
    if (capa->Imaging->XAddr != NULL) {
        device->capabilities.Imaging = malloc(sizeof(struct tt__ImagingCapabilities));
        device->capabilities.Imaging->XAddr = malloc(strlen(capa->Imaging->XAddr)+1);
        bzero(device->capabilities.Imaging->XAddr, strlen(capa->Imaging->XAddr)+1);
        memcpy(device->capabilities.Imaging->XAddr, capa->Imaging->XAddr, strlen(capa->Imaging->XAddr));
    }
    
    if (capa->Media->XAddr != NULL) {
        device->capabilities.Media = malloc(sizeof(struct tt__MediaCapabilities));
        device->capabilities.Media->XAddr = malloc(strlen(capa->Media->XAddr)+1);
        bzero(device->capabilities.Media->XAddr, strlen(capa->Media->XAddr)+1);
        memcpy(device->capabilities.Media->XAddr, capa->Media->XAddr, strlen(capa->Media->XAddr));
    }
    
    if (capa->PTZ->XAddr != NULL) {
        device->capabilities.PTZ = malloc(sizeof(struct tt__PTZCapabilities));
        device->capabilities.PTZ->XAddr = malloc(strlen(capa->PTZ->XAddr)+1);
        bzero(device->capabilities.PTZ->XAddr, strlen(capa->PTZ->XAddr)+1);
        memcpy(device->capabilities.PTZ->XAddr, capa->PTZ->XAddr, strlen(capa->PTZ->XAddr));
    }
}

void removeCapabilities(oc_device_t *device)
{
    if (device->capabilities.Analytics != NULL) {
        if (device->capabilities.Analytics->XAddr != NULL) {
            free(device->capabilities.Analytics->XAddr);
            device->capabilities.Analytics->XAddr = NULL;
        }
        free(device->capabilities.Analytics);
        device->capabilities.Analytics = NULL;
    }
    if (device->capabilities.Device != NULL) {
        if (device->capabilities.Device->XAddr != NULL) {
            free(device->capabilities.Device->XAddr);
            device->capabilities.Device->XAddr = NULL;
        }
        free(device->capabilities.Device);
        device->capabilities.Device = NULL;
    }
    if (device->capabilities.Events != NULL) {
        if (device->capabilities.Events->XAddr != NULL) {
            free(device->capabilities.Events->XAddr);
            device->capabilities.Events->XAddr = NULL;
        }
        free(device->capabilities.Events);
        device->capabilities.Events = NULL;
    }
    if (device->capabilities.Imaging != NULL) {
        if (device->capabilities.Imaging->XAddr != NULL) {
            free(device->capabilities.Imaging->XAddr);
            device->capabilities.Imaging->XAddr = NULL;
        }
        free(device->capabilities.Imaging);
        device->capabilities.Imaging = NULL;
    }
    if (device->capabilities.Media != NULL) {
        if (device->capabilities.Media->XAddr != NULL) {
            free(device->capabilities.Media->XAddr);
            device->capabilities.Media->XAddr = NULL;
        }
        free(device->capabilities.Media);
        device->capabilities.Media = NULL;
    }
    if (device->capabilities.PTZ != NULL) {
        if (device->capabilities.PTZ->XAddr != NULL) {
            free(device->capabilities.PTZ->XAddr);
            device->capabilities.PTZ->XAddr = NULL;
        }
        free(device->capabilities.PTZ);
        device->capabilities.PTZ = NULL;
    }
    if (device->capabilities.Extension != NULL) {
        free(device->capabilities.Extension);
        device->capabilities.Extension = NULL;
    }
    memset(&device->capabilities, 0, sizeof(device->capabilities));
}


void setProfiles(oc_device_t *device, struct _trt__GetProfilesResponse *profiles)
{
    if (device == NULL || profiles == NULL) {
        return;
    }
    
    removeProfiles(device);
    device->profiles.__sizeProfiles = profiles->__sizeProfiles;
    int i = 0;
    device->profiles.Profiles = malloc(sizeof(struct tt__Profile) * profiles->__sizeProfiles);
    for (i = 0; i < profiles->__sizeProfiles; i++) {
        struct tt__Profile *profile = &(device->profiles.Profiles[i]);
        memcpy(profile, &(profiles->Profiles[i]), sizeof(struct tt__Profile));
        
        profile->Name = malloc(strlen(profiles->Profiles[i].Name) + 1);
        bzero(profile->Name, strlen(profiles->Profiles[i].Name) + 1);
        memcpy(profile->Name, profiles->Profiles[i].Name, strlen(profiles->Profiles[i].Name));
        
        profile->token = malloc(strlen(profiles->Profiles[i].token) + 1);
        bzero(profile->token, strlen(profiles->Profiles[i].token) + 1);
        memcpy(profile->token, profiles->Profiles[i].token, strlen(profiles->Profiles[i].token));
        
        if (profiles->Profiles[i].PTZConfiguration != NULL) {
            profile->PTZConfiguration = malloc(sizeof(struct tt__PTZConfiguration));
            memcpy(profile->PTZConfiguration, profiles->Profiles[i].PTZConfiguration, sizeof(struct tt__PTZConfiguration));
            profile->PTZConfiguration->Name = malloc(strlen(profiles->Profiles[i].PTZConfiguration->Name) + 1);
            bzero(profile->PTZConfiguration->Name, strlen(profiles->Profiles[i].PTZConfiguration->Name) + 1);
            memcpy(profile->PTZConfiguration->Name, profiles->Profiles[i].PTZConfiguration->Name, strlen(profiles->Profiles[i].PTZConfiguration->Name));
            profile->PTZConfiguration->token = malloc(strlen(profiles->Profiles[i].PTZConfiguration->token) + 1);
            bzero(profile->PTZConfiguration->token, strlen(profiles->Profiles[i].PTZConfiguration->token) + 1);
            memcpy(profile->PTZConfiguration->token, profiles->Profiles[i].PTZConfiguration->token, strlen(profiles->Profiles[i].PTZConfiguration->token));
        }
    }
}

void removeProfiles(oc_device_t *device)
{
    if (device == NULL || device->profiles.__sizeProfiles == 0) {
        return;
    }
    int i = 0;
    for (i = 0; i < device->profiles.__sizeProfiles; i++) {
        struct tt__Profile profile = device->profiles.Profiles[i];
        if (profile.Name != NULL) {
            free(profile.Name);
        }
        if (profile.token != NULL) {
            free(profile.token);
        }
        if (profile.PTZConfiguration != NULL) {
            if (profile.PTZConfiguration->Name != NULL) {
                free(profile.PTZConfiguration->Name);
            }
            if (profile.PTZConfiguration->token != NULL) {
                free(profile.PTZConfiguration->token);
            }
            free(profile.PTZConfiguration);
        }
    }
    free(device->profiles.Profiles);
    device->profiles.Profiles = NULL;
    device->profiles.__sizeProfiles = 0;
}
~~~

#### 二、ONVIF控制，代码主要参照上面教程简单修改
OnvifControl.h
~~~
#ifndef OnvifControl_h
#define OnvifControl_h

#include "OnvifDevice.h"

#ifdef __cplusplus
extern "C"{
#endif

/// 搜索设备
/// @param userInfo 传入传出的指针
void ONVIF_DetectDevice(void (*cb)(void *userinfo, struct wsdd__ProbeMatchesType *wsdd__ProbeMatches), void *userInfo);


/// 获取设备信息
int ONVIF_GetDeviceInformation(oc_device_t *device);

/// 获取设备服务地址
int ONVIF_GetCapabilities(oc_device_t *device);


/// 获取设备profile
/// @param device 设备结构体
int ONVIF_GetProfile(oc_device_t *device);


int ONVIF_GetStreamUri(oc_device_t *device, int profileIndex, char *ioUri, unsigned int sizeuri);


int ONVIF_PTZGetStatus(oc_device_t *device, struct tt__Vector2D *ioPosition, enum xsd__boolean *moving);

/// ptz 方向控制
/// @param device 设备
/// @param direction 方向，0 左 1右  2上  3下
int ONVIF_PTZRelativeMove(oc_device_t *device, int direction);

#ifdef __cplusplus
}
#endif
    
#endif /* OnvifControl_h */
~~~

~~~
#include "OnvifControl.h"

#define SOAP_ASSERT     assert
#define SOAP_DBGLOG     printf
#define SOAP_DBGERR     printf

#define SOAP_TO         "urn:schemas-xmlsoap-org:ws:2005:04:discovery"
#define SOAP_ACTION     "http://schemas.xmlsoap.org/ws/2005/04/discovery/Probe"

#define SOAP_MCAST_ADDR "soap.udp://239.255.255.250:3702"                       // onvif规定的组播地址

#define SOAP_ITEM       ""                                                      // 寻找的设备范围
#define SOAP_TYPES      "dn:NetworkVideoTransmitter"                            // 寻找的设备类型

#define SOAP_SOCK_TIMEOUT    (10)   // socket超时时间（单秒秒）

#define SOAP_CHECK_ERROR(result, soap, str) \
do { \
if (SOAP_OK != (result) || SOAP_OK != (soap)->error) { \
soap_perror((soap), (str)); \
if (SOAP_OK == (result)) { \
(result) = (soap)->error; \
} \
goto EXIT; \
} \
} while (0)

void uriWithAuth(const char *orignalUri, const char *userName, const char *password, char *ioAuthURI)
{
    char *i = strchr(orignalUri, ':');
    long length = i - orignalUri + 3;
    memcpy(ioAuthURI, orignalUri, length);
    sprintf(ioAuthURI+length, "%s:%s@", userName, password);
    strcat(ioAuthURI, orignalUri + length);
    printf("%s %s\n", ioAuthURI, i);
}

void soap_perror(struct soap *soap, const char *str)
{
    if (NULL == str) {
        SOAP_DBGERR("[soap] error: %d, %s, %s\n", soap->error, *soap_faultcode(soap), *soap_faultstring(soap));
    } else {
        SOAP_DBGERR("[soap] %s error: %d, %s, %s\n", str, soap->error, *soap_faultcode(soap), *soap_faultstring(soap));
    }
    return;
}

void* ONVIF_soap_malloc(struct soap *soap, unsigned int n)
{
    void *p = NULL;
    
    if (n > 0) {
        p = soap_malloc(soap, n);
        SOAP_ASSERT(NULL != p);
        memset(p, 0x00 ,n);
    }
    return p;
}

struct soap *ONVIF_soap_new(int timeout)
{
    struct soap *soap = NULL;                                                   // soap环境变量
    
    SOAP_ASSERT(NULL != (soap = soap_new()));
    
    soap_set_namespaces(soap, namespaces);                                      // 设置soap的namespaces
    soap->recv_timeout    = timeout;                                            // 设置超时（超过指定时间没有数据就退出）
    soap->send_timeout    = timeout;
    soap->connect_timeout = timeout;
    
#if defined(__linux__) || defined(__linux)                                      // 参考https://www.genivia.com/dev.html#client-c的修改：
    soap->socket_flags = MSG_NOSIGNAL;                                          // To prevent connection reset errors
#endif
    
    soap_set_mode(soap, SOAP_C_UTFSTRING);                                      // 设置为UTF-8编码，否则叠加中文OSD会乱码
    
    return soap;
}

void ONVIF_soap_delete(struct soap *soap)
{
    soap_destroy(soap);                                                         // remove deserialized class instances (C++ only)
    soap_end(soap);                                                             // Clean up deserialized data (except class instances) and temporary data
    soap_done(soap);                                                            // Reset, close communications, and remove callbacks
    soap_free(soap);                                                            // Reset and deallocate the context created with soap_new or soap_copy
}

void ONVIF_init_header(struct soap *soap)
{
    struct SOAP_ENV__Header *header = NULL;
    
    SOAP_ASSERT(NULL != soap);
    
    header = (struct SOAP_ENV__Header *)ONVIF_soap_malloc(soap, sizeof(struct SOAP_ENV__Header));
    soap_default_SOAP_ENV__Header(soap, header);
    header->wsa__MessageID = (char*)soap_wsa_rand_uuid(soap);
    header->wsa__To        = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_TO) + 1);
    header->wsa__Action    = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_ACTION) + 1);
    strcpy(header->wsa__To, SOAP_TO);
    strcpy(header->wsa__Action, SOAP_ACTION);
    soap->header = header;
    
    return;
}

void ONVIF_init_ProbeType(struct soap *soap, struct wsdd__ProbeType *probe)
{
    struct wsdd__ScopesType *scope = NULL;                                      // 用于描述查找哪类的Web服务
    
    SOAP_ASSERT(NULL != soap);
    SOAP_ASSERT(NULL != probe);
    
    scope = (struct wsdd__ScopesType *)ONVIF_soap_malloc(soap, sizeof(struct wsdd__ScopesType));
    soap_default_wsdd__ScopesType(soap, scope);                                 // 设置寻找设备的范围
    scope->__item = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_ITEM) + 1);
    strcpy(scope->__item, SOAP_ITEM);
    
    memset(probe, 0x00, sizeof(struct wsdd__ProbeType));
    soap_default_wsdd__ProbeType(soap, probe);
    probe->Scopes = scope;
    probe->Types  = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_TYPES) + 1);     // 设置寻找设备的类型
    strcpy(probe->Types, SOAP_TYPES);
    
    return;
}

static int ONVIF_SetAuthInfo(struct soap *soap, const char *username, const char *password)
{
    int result = 0;
    
    SOAP_ASSERT(NULL != username);
    SOAP_ASSERT(NULL != password);
    
    result = soap_wsse_add_UsernameTokenDigest(soap, NULL, username, password);
    SOAP_CHECK_ERROR(result, soap, "add_UsernameTokenDigest");
    
EXIT:
    
    return result;
}



void ONVIF_DetectDevice(void (*cb)(void *userinfo, struct wsdd__ProbeMatchesType *wsdd__ProbeMatches), void *userInfo)
{
  
    int result = 0;
    unsigned int count = 0;                                                     // 搜索到的设备个数
    struct soap *soap = NULL;                                                   // soap环境变量
    struct wsdd__ProbeType      req;                                            // 用于发送Probe消息
    struct __wsdd__ProbeMatches rep;                                            // 用于接收Probe应答
//    struct wsdd__ProbeMatchType *probeMatch;
    
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(5)));
    
    ONVIF_init_header(soap);                                                    // 设置消息头描述
    ONVIF_init_ProbeType(soap, &req);                                           // 设置寻找的设备的范围和类型
    result = soap_send___wsdd__Probe(soap, SOAP_MCAST_ADDR, NULL, &req);        // 向组播地址广播Probe消息
    while (SOAP_OK == result)                                                   // 开始循环接收设备发送过来的消息
    {
        memset(&rep, 0x00, sizeof(rep));
        result = soap_recv___wsdd__ProbeMatches(soap, &rep);
        if (SOAP_OK == result) {
            if (soap->error) {
                soap_perror(soap, "ProbeMatches");
                if (NULL != cb) {
                    cb(userInfo, NULL);
                }
            } else {                                                            // 成功接收到设备的应答消息
                if (cb) {
                    cb(userInfo, rep.wsdd__ProbeMatches);
                }
            }
        } else if (soap->error) {
            if (NULL != cb) {
                cb(userInfo, NULL);
            }
            break;
        }
    }
    
    SOAP_DBGLOG("\ndetect end! It has detected %d devices!\n", count);
    
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    
    return ;
}

int ONVIF_GetDeviceInformation(oc_device_t *device)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _tds__GetDeviceInformation           devinfo_req;
    struct _tds__GetDeviceInformationResponse   devinfo_resp;
    
    SOAP_ASSERT(NULL != device->xAddr);
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    
    memset(&devinfo_req, 0x00, sizeof(devinfo_req));
    memset(&devinfo_resp, 0x00, sizeof(struct _tds__GetDeviceInformationResponse));
    result = soap_call___tds__GetDeviceInformation(soap, device->xAddr, NULL, &devinfo_req, &devinfo_resp);
    SOAP_CHECK_ERROR(result, soap, "GetDeviceInformation");
    
    if (result == SOAP_OK) {
        setInfo(device, &devinfo_resp);
    }
    
EXIT:
    
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    return result;
}

int ONVIF_GetCapabilities(oc_device_t *device)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _tds__GetCapabilities            req;
    struct _tds__GetCapabilitiesResponse    rep;
    
    SOAP_ASSERT(NULL != device->xAddr);
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    
    memset(&req, 0x00, sizeof(req));
    memset(&rep, 0x00, sizeof(rep));
    result = soap_call___tds__GetCapabilities(soap, device->xAddr, NULL, &req, &rep);
    SOAP_CHECK_ERROR(result, soap, "GetCapabilities");
    
    if (result == SOAP_OK) {
        setCapabilities(device, rep.Capabilities);
    }
    
EXIT:
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    return result;
}


int ONVIF_GetProfile(oc_device_t *device)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _trt__GetProfiles                req;
    struct _trt__GetProfilesResponse        resp;
    
    SOAP_ASSERT(NULL != device->capabilities.Media->XAddr);
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    
    memset(&req, 0x00, sizeof(req));
    memset(&resp, 0x00, sizeof(resp));
    
    result = SOAP_FMAC6 soap_call___trt__GetProfiles(soap, device->capabilities.Media->XAddr, NULL, &req, &resp);
    SOAP_CHECK_ERROR(result, soap, "GetProfile");
    if (result == SOAP_OK) {
        setProfiles(device, &resp);
    }

EXIT:
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    return result;
}


int ONVIF_GetStreamUri(oc_device_t *device, int profileIndex, char *ioUri, unsigned int sizeuri)
{
    int result = 0;
    struct soap *soap = NULL;
    struct tt__StreamSetup              ttStreamSetup;
    struct tt__Transport                ttTransport;
    struct _trt__GetStreamUri           req;
    struct _trt__GetStreamUriResponse   rep;
    
    SOAP_ASSERT(NULL != device->capabilities.Media->XAddr);
    SOAP_ASSERT(profileIndex < device->profiles.__sizeProfiles);
    SOAP_ASSERT(NULL != device->profiles.Profiles[profileIndex].token);
    SOAP_ASSERT(NULL != ioUri);
    memset(ioUri, 0x00, sizeuri);
    
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    memset(&req, 0x00, sizeof(req));
    memset(&rep, 0x00, sizeof(rep));
    memset(&ttStreamSetup, 0x00, sizeof(ttStreamSetup));
    memset(&ttTransport, 0x00, sizeof(ttTransport));
    ttStreamSetup.Stream                = tt__StreamType__RTP_Unicast;
    ttStreamSetup.Transport             = &ttTransport;
    ttStreamSetup.Transport->Protocol   = tt__TransportProtocol__RTSP;
    ttStreamSetup.Transport->Tunnel     = NULL;
    req.StreamSetup                     = &ttStreamSetup;
    req.ProfileToken                    = device->profiles.Profiles[profileIndex].token;
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    result = soap_call___trt__GetStreamUri(soap, device->capabilities.Media->XAddr, NULL, &req, &rep);
    SOAP_CHECK_ERROR(result, soap, "GetServices");
    
    result = -1;
    if (NULL != rep.MediaUri) {
        if (NULL != rep.MediaUri->Uri) {
            if (sizeuri > strlen(rep.MediaUri->Uri)) {
                uriWithAuth(rep.MediaUri->Uri, device->userName, device->password, ioUri);
                result = 0;
            } else {
                SOAP_DBGERR("Not enough cache!\n");
            }
        }
    }
    
EXIT:
    
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    
    return result;
}

int ONVIF_PTZGetStatus(oc_device_t *device, struct tt__Vector2D *ioPosition, enum xsd__boolean *moving)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _tptz__GetStatus           req;
    struct _tptz__GetStatusResponse   rep;
    
    SOAP_ASSERT(NULL != device->capabilities.PTZ->XAddr);
    SOAP_ASSERT(NULL != device->profiles.Profiles[0].PTZConfiguration->token);
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    memset(&req, 0x00, sizeof(req));
    memset(&rep, 0x00, sizeof(rep));
    req.ProfileToken = device->profiles.Profiles[0].PTZConfiguration->token;
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    result = soap_call___tptz__GetStatus(soap, device->capabilities.PTZ->XAddr, NULL, &req, &rep);
    SOAP_CHECK_ERROR(result, soap, "GetVideoEncoderConfigurationOptions");
    
    if (result == SOAP_OK) {
        if (rep.PTZStatus != NULL && rep.PTZStatus->Position != NULL ) {
            ioPosition->x = rep.PTZStatus->Position->PanTilt->x;
            ioPosition->y = rep.PTZStatus->Position->PanTilt->y;
        }
        if (rep.PTZStatus != NULL && rep.PTZStatus->MoveStatus != NULL) {
            *moving = (rep.PTZStatus->MoveStatus->PanTilt || rep.PTZStatus->MoveStatus->Zoom);
        }
    }
    
EXIT:
    
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    
    return result;
}

int ONVIF_PTZRelativeMove(oc_device_t *device, int direction)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _tptz__AbsoluteMove           req;
    struct _tptz__AbsoluteMoveResponse   rep;
    
    SOAP_ASSERT(NULL != device);
    SOAP_ASSERT(NULL != device->capabilities.PTZ->XAddr);
    SOAP_ASSERT(NULL != device->profiles.Profiles[0].PTZConfiguration->token);
    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));
    
    memset(&req, 0x00, sizeof(req));
    memset(&rep, 0x00, sizeof(rep));
    
    struct tt__Vector2D position = {0};
    enum xsd__boolean moving = xsd__boolean__false_;
    ONVIF_PTZGetStatus(device, &position, &moving);
    if (moving) {
        return -1;
    }
    
    printf("ptz position: x = %.1f, y = %.1f\n", position.x, position.y);
    
    struct tt__PTZVector Translation = {0};
    /** Optional element 'tptz:Speed' of XML schema type 'tt:PTZSpeed' */
    struct tt__PTZSpeed Speed = {0};
    
    req.ProfileToken = device->profiles.Profiles[0].PTZConfiguration->token;
    struct tt__Vector2D PanTilt = {0};
    switch (direction) {
        case 0:
            PanTilt.x = position.x + 0.1;
            break;
        case 1:
            PanTilt.x = position.x - 0.1;
            break;
        case 2:
            PanTilt.y = position.y + 0.1;
            break;
        case 3:
            PanTilt.y = position.y - 0.1;
            break;
        default:
            break;
    }
    Translation.PanTilt = &PanTilt;
    req.Position= &Translation;
    
    struct tt__Vector2D speed = {0};
    speed.x = 1;
    speed.y = 1;
    Speed.PanTilt = &speed;
    req.Speed = &Speed;
    
    ONVIF_SetAuthInfo(soap, device->userName, device->password);
    result = soap_call___tptz__AbsoluteMove(soap, device->capabilities.PTZ->XAddr, NULL, &req, &rep);
    SOAP_CHECK_ERROR(result, soap, "GetVideoEncoderConfigurationOptions");
    
EXIT:
    
    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    
    return result;
}
~~~

* 不知道是不是由于所使用的ipc是杂牌的原因，**ONVIF_PTZGetStatus** 无法获取到ipc当前ptz的准确偏移位置，所有实现准确的移动。



