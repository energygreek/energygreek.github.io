---
draft: true
date: 2021-04-14
tags: [ddns]
---

## code

```python
    """
    更新
    """
    parser = ArgumentParser(description=__description__,
                            epilog=__doc__, formatter_class=RawTextHelpFormatter)
    parser.add_argument('-v', '--version',
                        action='version', version=__version__)
    parser.add_argument('-c', '--config',
                        default="config.json", help="run with config file [配置文件路径]")
    config_file = parser.parse_args().config
    get_config(path=config_file)
    # Dynamicly import the dns module as configuration
    dns_provider = str(get_config('dns', 'dnspod').lower())
    dns = getattr(__import__('dns', fromlist=[dns_provider]), dns_provider)
    dns.Config.ID = get_config('id')
    dns.Config.TOKEN = get_config('token')
    dns.Config.TTL = get_config('ttl')
    if get_config('debug'):
        ip.DEBUG = get_config('debug')
        basicConfig(
            level=DEBUG,
            format='%(asctime)s <%(module)s.%(funcName)s> %(lineno)d@%(pathname)s \n[%(levelname)s] %(message)s')
        print("DDNS[", __version__, "] run:", os_name, sys.platform)
        print("Configuration was loaded from <==", path.abspath(config_file))
        print("=" * 25, ctime(), "=" * 25, sep=' ')

    proxy = get_config('proxy') or 'DIRECT'
    proxy_list = proxy.strip('; ') .split(';')

    cache = get_config('cache', True) and Cache(CACHE_FILE)
    if cache is False:
        info("Cache is disabled!")
    elif get_config.time >= cache.time:
        warning("Cache file is out of dated.")
        cache.clear()
    elif not cache:
        debug("Cache is empty.")
    update_ip('4', cache, dns, proxy_list)
    update_ip('6', cache, dns, proxy_list)


if __name__ == '__main__':
    main()

```

```
{
  "$schema": "https://ddns.newfuture.cc/schema/v2.8.json",
  "id": "",
  "token": "",
  "dns": "alidns",
  "ipv4": ["", ""],
  "index4": "public",
  "ttl": 600,
  "proxy": "DIRECT",
  "debug": false
}
```
