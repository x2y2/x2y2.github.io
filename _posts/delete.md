---
layout: post
title: Prometheus源码解析-Discovery
subtitle: Prometheus服务发现
date: 2020-05-12
categories: Go
cover: 'assets/img/road_green_trees_forest.jpg'
tags: prometheus discovery go
---

>prometheus项目的入口: github.com/prometheus/prometheus/cmd/prometheus/main.go, prometheus用oklog/run进行go routine编排，一个go routine称为一个actor, 一组actor为一个Groups。通过Add()方法将actor加入Groups，最后通过Run()方法启动所有的actor.

## 服务发现
>prometheus有2个地方用了服务发现机制，一个是数据采集，一个是告警通知。分别对应discoveryManagerScrape和scrapeManager两个变量，这两个变量都在cmd/prometheus/main.go，本文分析数据采集的服务发现机制。

### 结构体初始化
定义变量scrapre Manager结构体初始化，discoveryManagerScrape是一个scrape Manager结构体对象
```discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))
```
### 配置文件加载初始化
main.reloaders 是一个函数类型的slice,其中包含discoveryManagerScrape.ApplyConfig

```
func(cfg *config.Config) error {
    c := make(map[string]sd_config.ServiceDiscoveryConfig)
    for _, v := range cfg.ScrapeConfigs {
        c[v.JobName] = v.ServiceDiscoveryConfig
    }
    return discoveryManagerScrape.ApplyConfig(c)
}
```

当执行g.Run()的时候，所有的actor都作为goroutine执行
执行调用reloadConfig函数时会将reloads作为参数传入
```
if err := reloadConfig(cfg.configFile, logger, reloaders...); err != nil {
    return errors.Wrapf(err, "error loading config from %q", cfg.configFile)
}
```
在main.reloadConfig 中遍历rls,就是对reloaders中的函数逐个执行
```
for _, rl := range rls {
    if err := rl(conf); err != nil {
        level.Error(logger).Log("msg", "Failed to apply configuration", "err", err)
        failed = true
    }
}
```
执行discoveryManagerScrape.ApplyConfig(c)即discovery.manager.ApplyConfig(c)，在这个方法中会调用registerProviders，他的作用是将配置文件中scrape使用的服务发现方式,保存为provider对象，最后会存到[]\*provider中。对应到Manager结构体的providers []\*provider字段。
provider结构体
```
type provider struct {
    name   string
    d      Discoverer
    subs   []string
    config interface{}
}
```

```
for name, scfg := range cfg {
    failedCount += m.registerProviders(scfg, name)
    discoveredTargets.WithLabelValues(m.name, name).Set(0)
}
```
在registerProviders方法中实现了一个add函数，作用是将scrape的所有服务发现方式封装为provider对象。从add函数中可以看出所有的服务发现机制都要实现Discoverer接口。其中的d就是实现了Discoverer接口的结构体

```
add := func(cfg interface{}, newDiscoverer func() (Discoverer, error)){
    d, err := newDiscoverer()
    provider := provider{
        name:   fmt.Sprintf("%s/%d", t, len(m.providers)),
        d:      d,
        config: cfg,
        subs:   []string{setName},
    }
    m.providers = append(m.providers, &provider)
}

for _, c := range cfg.ConsulSDConfigs {
    add(c, func() (Discoverer, error) {
        return consul.NewDiscovery(c, log.With(m.logger, "discovery", "consul"))
    })
}
```
Discoverer接口
```
type Discoverer interface {
    Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```
至此，scrape的服务发现注册完成，接下来执行provider从服务注册中心，获取服务。
```
for _, prov := range m.providers {
    m.startProvider(m.ctx, prov)
}
```
这里主要是获取服务，go p.d.Run(ctx, updates)是执行provider结构体的Run方法，本文以Consul作为服务发现的列子。
```
func (m *Manager) startProvider(ctx context.Context, p *provider) {
    level.Debug(m.logger).Log("msg", "Starting provider", "provider", p.name, "subs", fmt.Sprintf("%v", p.subs))
    ctx, cancel := context.WithCancel(ctx)
    updates := make(chan []*targetgroup.Group)

    m.discoverCancel = append(m.discoverCancel, cancel)

    go p.d.Run(ctx, updates)
    go m.updater(ctx, p, updates)
}
```
discovery.consul.Discovery.Run




