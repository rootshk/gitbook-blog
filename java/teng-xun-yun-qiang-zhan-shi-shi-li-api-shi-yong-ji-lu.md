# \[腾讯云] 抢占式实例API使用记录

> 1. 最近了解到阿里云和腾讯云的抢占式实例很是便宜，感觉可以研究研究，使用场景，比如说压测、梯子、临时扩容
> 2. 对比了一下两家的价格，以2C2G的配置，腾讯云¥0.040/小时(¥29.76/月)、阿里云¥0.033/小时(突发性能实例t6)(¥24.552/月)。
> 3. 门槛：阿里云余额需要>¥100, 腾讯云无限制
> 4. 所以选择腾讯云作为目标

## 前提

### 注册腾讯云

### 获取secretId和secretKey

## 具体流程

* 获取所有可用区
* 遍历可用区内的全部可用的抢占式实例
* 遍历获取全部抢占式实例价格同步到数据库

## 代码

* pom.xml

```
<dependencies>
        <dependency>
            <groupId>com.tencentcloudapi</groupId>
            <artifactId>tencentcloud-sdk-java</artifactId>
            <version>4.0.11</version>
        </dependency>
</dependencies>
```

* TencentCloudUtil.java

```
package xxx.util;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import com.tencentcloudapi.common.Credential;
import com.tencentcloudapi.common.profile.ClientProfile;
import com.tencentcloudapi.common.profile.HttpProfile;
import com.tencentcloudapi.common.exception.TencentCloudSDKException;
import com.tencentcloudapi.cvm.v20170312.CvmClient;
import com.tencentcloudapi.cvm.v20170312.models.*;
import jodd.http.HttpRequest;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;


/**
 * @author hongkeng
 */
@Slf4j
public class TencentCloudUtil {

    private static final String ENDPOINT = "cvm.tencentcloudapi.com";
    public static final String SECRET_ID = "";
    public static final String SECRET_KEY = "";

    /**
     * 网络计费类型
     */
    public enum InternetChargeType {
        BANDWIDTH_PREPAID(0, "预付费按带宽结算"),
        TRAFFIC_POSTPAID_BY_HOUR(1, "流量按小时后付费"),
        BANDWIDTH_POSTPAID_BY_HOUR(2, "带宽按小时后付费"),
        BANDWIDTH_PACKAGE(3, "带宽包用户");

        private final int code;
        private final String cname;

        InternetChargeType(final int code, final String cname) {
            this.code = code;
            this.cname = cname;
        }

        public int getCode() {
            return code;
        }

        public String getCName() {
            return cname;
        }
    }

    /**
     * 实例计费类型
     */
    public enum InstanceChargeType {
        /**
         * 预付费，即包年包月
         */
        PREPAID(0, "预付费，即包年包月"),
        /**
         * 按小时后付费
         */
        POSTPAID_BY_HOUR(1, "按小时后付费"),
        /**
         * 独享子机
         */
        CDHPAID(2, "独享子机"),
        /**
         * 竞价付费
         */
        SPOTPAID(3, "竞价付费"),
        /**
         * 专用集群付费
         */
        CDCPAID(4, "专用集群付费");

        private final int code;
        private final String cname;

        InstanceChargeType(final int code, final String cname) {
            this.code = code;
            this.cname = cname;
        }

        public int getCode() {
            return code;
        }

        public String getCName() {
            return cname;
        }
    }

    public static List<InstanceInfo> getTotalInfo() {
        List<InstanceInfo> l = new ArrayList<>();
        List<RegionInfo> regionInfos = regionInfos(SECRET_ID, SECRET_KEY);
        regionInfos.forEach(r -> {
            log.info("获取到地域: {}/{}", r.getRegion(), r.getRegionName());
            List<InstanceTypeConfig> cs = describeInstanceTypeConfigs(SECRET_ID, SECRET_KEY, r.getRegion());
            cs.forEach(c -> {
                log.info("可用机型: {}", c.getInstanceType());
                l.add(InstanceInfo.builder().r(r).instanceTypeConfig(c).build());
            });
        });
        return l;
    }

    private static final String TERMINATION_URL = "http://metadata.tencentyun.com/latest/meta-data/spot/termination-time";

    /**
     * 查询是否快要回收
     * curl metadata.tencentyun.com/latest/meta-data/spot/termination-time
     */
    public static boolean termination() {
        try {
            // 2018-08-18 12:05:33
            String date = HttpRequest.get(TERMINATION_URL).send().bodyText();
            if (StringUtils.isBlank(date) || StringUtils.contains(date, "404")) {
                return false;
            }
            LocalDateTime localDateTime = DateUtil.parseYyyyMmDdHhMmSs(date);
            log.info("过期时间: {}", date);
            log.info("还有{}秒", LocalDateTime.now().until(localDateTime, ChronoUnit.SECONDS));
            return true;
        } catch (Exception e) {
            log.error("termination error", e);
            return false;
        }
    }

    /**
     * 创建实例
     *
     * @param secretId
     * @param secretKey
     * @param region
     * @param regionZone
     * @param instanceType
     * @param imageId
     * @param instanceChargeType
     * @param internetChargeType
     * @param maxBandwidthOut
     * @param publicIp
     * @param dryRun
     * @return
     */
    public static String[] createInstance(String secretId, String secretKey, String region, String regionZone, String instanceType, String imageId,
                                        InstanceChargeType instanceChargeType, InternetChargeType internetChargeType, Long maxBandwidthOut, Boolean publicIp, Boolean dryRun) {
        try{
            Credential cred = new Credential(secretId, secretKey);
            // 实例化一个http选项，可选的，没有特殊需求可以跳过
            HttpProfile httpProfile = new HttpProfile();
            httpProfile.setEndpoint(ENDPOINT);
            // 实例化一个client选项，可选的，没有特殊需求可以跳过
            ClientProfile clientProfile = new ClientProfile();
            clientProfile.setHttpProfile(httpProfile);
            // 实例化要请求产品的client对象,clientProfile是可选的
            CvmClient client = new CvmClient(cred, region, clientProfile);
            // 实例化一个请求对象,每个接口都会对应一个request对象
            RunInstancesRequest req = new RunInstancesRequest();
            req.setInstanceChargeType(instanceChargeType.name());
            Placement placement1 = new Placement();
            placement1.setZone(regionZone);
            req.setPlacement(placement1);

            req.setInstanceType(instanceType);
            req.setImageId(imageId);
            InternetAccessible internetAccessible1 = new InternetAccessible();
            internetAccessible1.setInternetChargeType(internetChargeType.name());
            internetAccessible1.setInternetMaxBandwidthOut(maxBandwidthOut);
            internetAccessible1.setPublicIpAssigned(publicIp);
            req.setInternetAccessible(internetAccessible1);

            req.setInstanceCount(1L);
            req.setInstanceName("hongkeng" + DateUtil.toDayS());
            LoginSettings loginSettings1 = new LoginSettings();
            loginSettings1.setKeyIds(new String[]{"skey-f3m2f9cn"});
//            loginSettings1.setPassword("Aliyun45685");
            req.setLoginSettings(loginSettings1);

            req.setHostName("hongkeng" + DateUtil.toDayS());
            req.setDryRun(dryRun);
            RunInstancesResponse resp = client.RunInstances(req);
            log.info("InstanceIds: {}", List.of(resp.getInstanceIdSet()));
            log.info("RequestId: {}", resp.getRequestId());
            return resp.getInstanceIdSet();
        } catch (TencentCloudSDKException e) {
            log.error("创建实例失败: ", e);
            return null;
        }
    }

    public static void updateRegionInfoPrice(InstanceInfo i) {
        RegionInfo r = i.getR();
        InstanceTypeConfig c = i.getInstanceTypeConfig();
        InstancePrice instancePrice = instancePrice(SECRET_ID, SECRET_KEY, r.getRegion(), c.getZone(), c.getInstanceType(), "img-l8og963d",
                InstanceChargeType.SPOTPAID, InternetChargeType.TRAFFIC_POSTPAID_BY_HOUR, 100L, true);
        i.setInstancePrice(instancePrice);
    }

    /**
     * 获取腾讯云可用地域
     *
     * @param secretId s
     * @param secretKey s
     * @return s
     */
    public static List<RegionInfo> regionInfos(String secretId, String secretKey) {
        try{
            // 实例化一个认证对象，入参需要传入腾讯云账户secretId，secretKey,此处还需注意密钥对的保密
            // 密钥可前往https://console.cloud.tencent.com/cam/capi网站进行获取
            Credential cred = new Credential(secretId, secretKey);
            // 实例化一个http选项，可选的，没有特殊需求可以跳过
            HttpProfile httpProfile = new HttpProfile();
            httpProfile.setEndpoint(ENDPOINT);
            // 实例化一个client选项，可选的，没有特殊需求可以跳过
            ClientProfile clientProfile = new ClientProfile();
            clientProfile.setHttpProfile(httpProfile);
            // 实例化要请求产品的client对象,clientProfile是可选的
            CvmClient client = new CvmClient(cred, "", clientProfile);
            // 实例化一个请求对象,每个接口都会对应一个request对象
            DescribeRegionsRequest req = new DescribeRegionsRequest();
            DescribeRegionsResponse resp = client.DescribeRegions(req);
            List<RegionInfo> infos = new ArrayList<>(10);
            for (RegionInfo r : resp.getRegionSet()) {
                if (!"AVAILABLE".equals(r.getRegionState())) {
                    continue;
                }
                infos.add(r);
            }
            return infos;
        } catch (TencentCloudSDKException e) {
            log.error("获取地域失败", e);
            return Collections.emptyList();
        }
    }

    public static Instance instanceInfo(String secretId, String secretKey, String region, String instanceId) {
        try{
            Credential cred = new Credential(secretId, secretKey);
            // 实例化一个http选项，可选的，没有特殊需求可以跳过
            HttpProfile httpProfile = new HttpProfile();
            httpProfile.setEndpoint(ENDPOINT);
            // 实例化一个client选项，可选的，没有特殊需求可以跳过
            ClientProfile clientProfile = new ClientProfile();
            clientProfile.setHttpProfile(httpProfile);
            // 实例化要请求产品的client对象,clientProfile是可选的
            CvmClient client = new CvmClient(cred, region, clientProfile);
            // 实例化一个请求对象,每个接口都会对应一个request对象
            DescribeInstancesRequest req = new DescribeInstancesRequest();
            String[] instanceIds1 = {instanceId};
            req.setInstanceIds(instanceIds1);
            // 返回的resp是一个DescribeInstancesResponse的实例，与请求对象对应
            DescribeInstancesResponse resp = client.DescribeInstances(req);
            Instance[] instanceSet = resp.getInstanceSet();
            if (instanceSet.length == 0) {
                return null;
            }
            return instanceSet[0];
        } catch (TencentCloudSDKException e) {
            log.error("获取机型详情失败", e);
            return null;
        }
    }

    /**
     * 获取某个地域的机型列表
     *
     * @param secretId
     * @param secretKey
     * @param region
     * @return
     */
    public static List<InstanceTypeConfig> describeInstanceTypeConfigs(String secretId, String secretKey, String region) {
        try{
            // 实例化一个认证对象，入参需要传入腾讯云账户secretId，secretKey,此处还需注意密钥对的保密
            // 密钥可前往https://console.cloud.tencent.com/cam/capi网站进行获取
            Credential cred = new Credential(secretId, secretKey);
            // 实例化一个http选项，可选的，没有特殊需求可以跳过
            HttpProfile httpProfile = new HttpProfile();
            httpProfile.setEndpoint(ENDPOINT);
            // 实例化一个client选项，可选的，没有特殊需求可以跳过
            ClientProfile clientProfile = new ClientProfile();
            clientProfile.setHttpProfile(httpProfile);
            // 实例化要请求产品的client对象,clientProfile是可选的
            CvmClient client = new CvmClient(cred, region, clientProfile);
            // 实例化一个请求对象,每个接口都会对应一个request对象
            DescribeInstanceTypeConfigsRequest req = new DescribeInstanceTypeConfigsRequest();

            // 返回的resp是一个DescribeInstanceTypeConfigsResponse的实例，与请求对象对应
            DescribeInstanceTypeConfigsResponse resp = client.DescribeInstanceTypeConfigs(req);
            return List.of(resp.getInstanceTypeConfigSet());
        } catch (TencentCloudSDKException e) {
            log.error("获取机型列表失败", e);
            return Collections.emptyList();
        }
    }

    public static InstancePrice instancePrice(String secretId, String secretKey, String region, String regionZone, String instanceType, String imageId,
                                           InstanceChargeType instanceChargeType, InternetChargeType internetChargeType, Long maxBandwidthOut, Boolean publicIp) {
        try{
            // 实例化一个认证对象，入参需要传入腾讯云账户secretId，secretKey,此处还需注意密钥对的保密
            // 密钥可前往https://console.cloud.tencent.com/cam/capi网站进行获取
            Credential cred = new Credential(secretId, secretKey);
            // 实例化一个http选项，可选的，没有特殊需求可以跳过
            HttpProfile httpProfile = new HttpProfile();
            httpProfile.setEndpoint(ENDPOINT);
            // 实例化一个client选项，可选的，没有特殊需求可以跳过
            ClientProfile clientProfile = new ClientProfile();
            clientProfile.setHttpProfile(httpProfile);
            // 实例化要请求产品的client对象,clientProfile是可选的
            CvmClient client = new CvmClient(cred, region, clientProfile);
            // 实例化一个请求对象,每个接口都会对应一个request对象
            InquiryPriceRunInstancesRequest req = new InquiryPriceRunInstancesRequest();
            req.setInstanceChargeType(instanceChargeType.name());
            Placement placement1 = new Placement();
            placement1.setZone(regionZone);
            req.setPlacement(placement1);

            req.setInstanceType(instanceType);
            // "img-l8og963d"
            req.setImageId(imageId);
            InternetAccessible internetAccessible1 = new InternetAccessible();
            internetAccessible1.setInternetChargeType(internetChargeType.name());
            internetAccessible1.setInternetMaxBandwidthOut(maxBandwidthOut);
            internetAccessible1.setPublicIpAssigned(publicIp);
            req.setInternetAccessible(internetAccessible1);

            // 返回的resp是一个InquiryPriceRunInstancesResponse的实例，与请求对象对应
            InquiryPriceRunInstancesResponse resp = client.InquiryPriceRunInstances(req);
            Float instancePrice = resp.getPrice().getInstancePrice().getUnitPriceDiscount();
            Float bandwidthPrice = resp.getPrice().getBandwidthPrice().getUnitPriceDiscount();
            return InstancePrice.builder()
                    .instancePrice(instancePrice == null ? BigDecimal.ZERO : BigDecimal.valueOf(instancePrice).setScale(3, RoundingMode.HALF_UP))
                    .bandwidthPrice(bandwidthPrice == null ? BigDecimal.ZERO : BigDecimal.valueOf(bandwidthPrice).setScale(3, RoundingMode.HALF_UP))
                    .build();
        } catch (TencentCloudSDKException e) {
            log.error("获取机型价格失败: {}", e.getMessage());
            return InstancePrice.defaultPrice(e.getMessage());
        }
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class InstancePrice {

        private BigDecimal instancePrice;

        private BigDecimal bandwidthPrice;

        private String message;

        public BigDecimal getTotalPrice() {
            BigDecimal a = instancePrice == null ? BigDecimal.ZERO : instancePrice;
            BigDecimal b = bandwidthPrice == null ? BigDecimal.ZERO : bandwidthPrice;
            return a.add(b);
        }

        public static InstancePrice defaultPrice(String failMessage) {
            return InstancePrice.builder().message(failMessage).instancePrice(BigDecimal.valueOf(-1)).bandwidthPrice(BigDecimal.valueOf(-1)).build();
        }
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class InstanceInfo {

        private RegionInfo r;

        private InstanceTypeConfig instanceTypeConfig;

        private InstancePrice instancePrice;

    }

}
```

* InstanceController.java

```
package xxx.controller;

import com.querydsl.core.types.dsl.BooleanExpression;
import com.tencentcloudapi.cvm.v20170312.models.Instance;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.util.Assert;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import xxx.entity.CloudInstance;
import xxx.entity.QCloudInstance;
import xxx.service.CloudInstanceService;
import xxx.util.TencentCloudUtil;

import javax.annotation.Resource;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

import static xxx.util.TencentCloudUtil.SECRET_ID;
import static xxx.util.TencentCloudUtil.SECRET_KEY;

/**
 * @author hongkeng
 */
@Slf4j
@Tag(name = "主机")
@RestController
@RequestMapping("/instance")
public class InstanceController {

    @Resource
    private CloudInstanceService cloudInstanceService;

    @RequestMapping("/updateInstance")
    public void updateInstance() {
        QCloudInstance q = CloudInstanceService.Q;
        List<TencentCloudUtil.InstanceInfo> list = TencentCloudUtil.getTotalInfo();
        list.forEach(l -> {
            CloudInstance i = cloudInstanceService.find(q.zone.eq(l.getInstanceTypeConfig().getZone())
                    .and(q.instanceFamily.eq(l.getInstanceTypeConfig().getInstanceFamily()))
                    .and(q.instanceType.eq(l.getInstanceTypeConfig().getInstanceType())));
            if (i == null) {
                i = CloudInstance.builder()
                        .region(l.getR().getRegion())
                        .regionName(l.getR().getRegionName())
                        .zone(l.getInstanceTypeConfig().getZone())
                        .instanceFamily(l.getInstanceTypeConfig().getInstanceFamily())
                        .instanceType(l.getInstanceTypeConfig().getInstanceType())
                        .cpu(l.getInstanceTypeConfig().getCPU())
                        .gpu(l.getInstanceTypeConfig().getGPU())
                        .memory(l.getInstanceTypeConfig().getMemory())
                        .build();
            } else {
                i.setRegion(l.getR().getRegion());
                i.setRegionName(l.getR().getRegionName());
                i.setCpu(l.getInstanceTypeConfig().getCPU());
                i.setGpu(l.getInstanceTypeConfig().getGPU());
                i.setMemory(l.getInstanceTypeConfig().getMemory());
            }
            cloudInstanceService.save(i);
        });
    }

    @RequestMapping("/updateInstancePrice/{id}")
    public void updateInstancePrice(@PathVariable Long id,
                                    @RequestParam(required = false, defaultValue = "img-l8og963d") String imageId,
                                    @RequestParam(required = false, defaultValue = "SPOTPAID") TencentCloudUtil.InstanceChargeType instanceChargeType,
                                    @RequestParam(required = false, defaultValue = "TRAFFIC_POSTPAID_BY_HOUR") TencentCloudUtil.InternetChargeType internetChargeType,
                                    @RequestParam(required = false, defaultValue = "100") Long maxBandwidthOut,
                                    @RequestParam(required = false, defaultValue = "true") Boolean publicIp) {
        CloudInstance l = cloudInstanceService.find(id);
        Assert.notNull(l, "未查询到id");
        TencentCloudUtil.InstancePrice price = TencentCloudUtil.instancePrice(SECRET_ID, SECRET_KEY, l.getRegion(), l.getZone(), l.getInstanceType(), imageId, instanceChargeType, internetChargeType, maxBandwidthOut, publicIp);
        if (price.getInstancePrice() == null || price.getInstancePrice().compareTo(BigDecimal.ZERO) <= 0
                || price.getBandwidthPrice() == null || price.getBandwidthPrice().compareTo(BigDecimal.ZERO) <= 0) {
            l.setFailDateTime(LocalDateTime.now());
            l.setFailMessage(price.getMessage());
            cloudInstanceService.save(l);
        }
        l.setFailMessage(null);
        l.setInstancePrice(price.getInstancePrice());
        l.setBandwidthPrice(price.getBandwidthPrice());
        cloudInstanceService.save(l);
    }

    @RequestMapping("/updateInstancePrice")
    public void updateInstancePrice(@RequestParam(required = false, defaultValue = "img-l8og963d") String imageId,
                                    @RequestParam(required = false, defaultValue = "SPOTPAID") TencentCloudUtil.InstanceChargeType instanceChargeType,
                                    @RequestParam(required = false, defaultValue = "TRAFFIC_POSTPAID_BY_HOUR") TencentCloudUtil.InternetChargeType internetChargeType,
                                    @RequestParam(required = false, defaultValue = "100") Long maxBandwidthOut,
                                    @RequestParam(required = false, defaultValue = "true") Boolean publicIp,
                                    @RequestParam(required = false, defaultValue = "true") Boolean handlerFail,
                                    @RequestParam(required = false) String region,
                                    @RequestParam(required = false) String zone,
                                    @RequestParam(required = false) String instanceFamily,
                                    @RequestParam(required = false) String instanceType,
                                    @RequestParam(required = false) Long maxCpu,
                                    @RequestParam(required = false) Long maxMem) {
        QCloudInstance q = CloudInstanceService.Q;
        BooleanExpression eq = q.failMessage.isNull().or(q.failMessage.notIn("The specified Zone, instanceType does not support for spot."));
        if (maxCpu != null && maxCpu > 0) {
            eq = eq.and(q.cpu.lt(maxCpu + 1));
        }
        if (maxMem != null && maxMem > 0) {
            eq = eq.and(q.memory.lt(maxMem + 1));
        }
        if (StringUtils.isNotBlank(region)) {
            eq = eq.and(q.region.eq(region));
        }
        if (StringUtils.isNotBlank(zone)) {
            eq = eq.and(q.zone.eq(zone));
        }
        if (StringUtils.isNotBlank(instanceFamily)) {
            eq = eq.and(q.instanceFamily.eq(instanceFamily));
        }
        if (StringUtils.isNotBlank(instanceType)) {
            eq = eq.and(q.instanceType.eq(instanceType));
        }
        if (handlerFail) {
            BooleanExpression e1 = q.instancePrice.isNull().or(q.bandwidthPrice.isNull());
            BooleanExpression e2 = q.failDateTime.isNull().or(q.failDateTime.before(LocalDateTime.now().plusHours(-1)));
            eq = eq.and(e1).and(e2);
        }
        List<CloudInstance> list = cloudInstanceService.findList(eq);
        for (CloudInstance l : list) {
            TencentCloudUtil.InstancePrice price = TencentCloudUtil.instancePrice(SECRET_ID, SECRET_KEY, l.getRegion(), l.getZone(), l.getInstanceType(), imageId, instanceChargeType, internetChargeType, maxBandwidthOut, publicIp);
            if (price.getInstancePrice() == null || price.getInstancePrice().compareTo(BigDecimal.ZERO) <= 0
                    || price.getBandwidthPrice() == null || price.getBandwidthPrice().compareTo(BigDecimal.ZERO) <= 0) {
                l.setFailDateTime(LocalDateTime.now());
                l.setFailMessage(price.getMessage());
                cloudInstanceService.save(l);
                continue;
            }
            l.setFailMessage(null);
            l.setInstancePrice(price.getInstancePrice());
            l.setBandwidthPrice(price.getBandwidthPrice());
            cloudInstanceService.save(l);
        }
    }

    @RequestMapping("/termination")
    public Boolean termination() {
        return TencentCloudUtil.termination();
    }

    @RequestMapping("/instanceInfo")
    public String instanceInfo(@RequestParam Long instanceId, @RequestParam String instanceIds) {
        CloudInstance instance = cloudInstanceService.find(instanceId);
        Assert.notNull(instance, "未查询到数据");
        Instance info = TencentCloudUtil.instanceInfo(SECRET_ID, SECRET_KEY, instance.getRegion(), instanceIds);
        Assert.notNull(info, "未查询到主机信息");
        String ip = info.getPublicIpAddresses().length == 0 ? "" : info.getPublicIpAddresses()[0];
        log.info("ssh root@{}", ip);
        return "ssh root@" + ip;
    }

    @RequestMapping("/createInstance")
    public String createInstance(@RequestParam Long instanceId,
                                       @RequestParam(required = false, defaultValue = "img-l8og963d") String imageId,
                                       @RequestParam(required = false, defaultValue = "SPOTPAID") TencentCloudUtil.InstanceChargeType instanceChargeType,
                                       @RequestParam(required = false, defaultValue = "TRAFFIC_POSTPAID_BY_HOUR") TencentCloudUtil.InternetChargeType internetChargeType,
                                       @RequestParam(required = false, defaultValue = "100") Long maxBandwidthOut,
                                       @RequestParam(required = false, defaultValue = "true") Boolean publicIp) throws InterruptedException {
        CloudInstance instance = cloudInstanceService.find(instanceId);
        Assert.notNull(instance, "未查询到数据");
        String[] result = TencentCloudUtil.createInstance(SECRET_ID, SECRET_KEY, instance.getRegion(), instance.getZone(),
                instance.getInstanceType(), imageId, instanceChargeType, internetChargeType,
                maxBandwidthOut, publicIp, false);
        String r = result == null || result.length == 0 ? null : result[0];
        log.info("instanceIds: {}", r);
        Thread.sleep(10 * 1000);
        return instanceInfo(instanceId, r);
    }

}
```

## 数据库查询最便宜主机

```
select id, concat(instance_price*24*31, '元/月') as monthPrice, region_name, zone, instance_family, instance_type, instance_price, cpu, memory
from cloud_instance 
WHERE instance_price is not Null 
order by
instance_price asc,
cpu desc,
memory desc,
region_name desc,
instance_family desc,
instance_type desc,
zone desc
LIMIT 0, 100;
```

![2022-05-06T06:43:37.png](https://roothk.top/usr/uploads/2022/05/3743201953.png)
