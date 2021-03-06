>* 原文链接 : [Geolocation using multiple services](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html)
* 原文作者 : [wsdookadr](https://github.com/wsdookadr)
* 译文出自 : [掘金翻译计划](https://github.com/xitu/gold-miner)
* 译者 : 
* 校对者:


## Intro

In a [previous post](https://blog.garage-coding.com/2015/12/24/out-on-the-streets.html) I wrote about [PostGIS](http://postgis.net/) and ways of querying geographical data. This post will focus on building a system that queries free geolocation services <sup>[1](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fn.1)</sup> and aggregates their results.

## Overview

In summary, we're making requests to different web services (or APIs), we're doing [reverse geocoding](https://en.wikipedia.org/wiki/Reverse_geocoding)on the results and then we'll aggregate them. ![](http://ac-Myg6wSTV.clouddn.com/2442a3bd132f453eb9eb.png)



## Comparing geonames and openstreetmap

To relate to the [previous post](https://blog.garage-coding.com/2015/12/24/out-on-the-streets.html), here are some differences between geonames and openstreetmap:

![](http://ww1.sinaimg.cn/large/a490147fgw1f5raumu7jtj20gw09ujt1.jpg)

They are meant for different purposes. Geonames is meant for city/administrative area/country data and can be used for [geocoding](http://www.geonames.org/export/geonames-search.html). Openstreetmap has much more detailed data (one could probably extract the geonames data using openstreetmap) and can be used for geocoding, route planning [and](http://wiki.openstreetmap.org/wiki/Applications_of_OpenStreetMap) [more](http://wiki.openstreetmap.org/wiki/List_of_OSM-based_services) .

## Asynchronous requests to geolocation services

We're using the [gevent](http://www.gevent.org/) library to make asynchronous requests to the geolocation services.

    import gevent
    import gevent.greenlet
    from gevent import monkey; gevent.monkey.patch_all()

    geoip_service_urls=[
            ['geoplugin'    , 'http://www.geoplugin.net/json.gp?ip={ip}' ],
            ['ip-api'       , 'http://ip-api.com/json/{ip}'              ],
            ['nekudo'       , 'https://geoip.nekudo.com/api/{ip}'        ],
            ['geoiplookup'  , 'http://api.geoiplookup.net/?query={ip}'   ],
            ]

    # fetch url in asynchronous mode (makes use of gevent)
    def fetch_url_async(url, tag, timeout=2.0):
        data = None
        try:
            opener = urllib2.build_opener(urllib2.HTTPSHandler())
            opener.addheaders = [('User-agent', 'Mozilla/')]
            urllib2.install_opener(opener)
            data = urllib2.urlopen(url,timeout=timeout).read()
        except Exception, e:
            pass

        return [tag, data]

    # expects req_data to be in this format: [ ['tag', url], ['tag', url], .. ]
    def fetch_multiple_urls_async(req_data):

        # start the threads (greenlets)
        threads_ = []
        for u in req_data:
            (tag, url) = u
            new_thread = gevent.spawn(fetch_url_async, url, tag)
            threads_.append(new_thread)

        # wait for threads to finish
        gevent.joinall(threads_)

        # retrieve threads return values
        results = []
        for t in threads_:
            results.append(t.get(block=True, timeout=5.0))

        return results

    def process_service_answers(location_data):
        # 1) extract lat/long data from responses
        # 2) reverse geocoding using geonames
        # 3) aggregate location data
        #    (for example, one way of doing this would
        #     be to choose the location that most services
        #     agree on)
        pass

    def geolocate_ip(ip):
        urls = []
        for grp in geoip_service_urls:
            tag, url = grp
            urls.append([tag, url.format(ip=ip)])
        results = fetch_multiple_urls_async(urls)
        answer = process_service_answers(results)
        return answer

## City name ambiguity

### Cities with the same name within the same country

There are many cities with the same name within a country, in different states/administrative regions. There's also cities with the same name in different countries. For example, according to Geonames, [there are](https://www.google.com/webhp?#q=%22united+states%22+(%22city%22+OR+%22town%22)+inurl:clinton+site:wikipedia.org) 24 cities named _Clinton_ in the US (in 23 different states, with two cities named _Clinton_ in the same state of Michigan).

    WITH duplicate_data AS (
        SELECT
        city_name,
        array_agg(ROW(country_code, region_code)) AS dupes
        FROM city_region_data
        WHERE country_code = 'US'
        GROUP BY city_name, country_code
        ORDER BY COUNT(ROW(country_code, region_code)) DESC
    )
    SELECT
    city_name,
    ARRAY_LENGTH(dupes, 1) AS duplicity,
    ( CASE WHEN ARRAY_LENGTH(dupes,1) > 9 
      THEN CONCAT(SUBSTRING(ARRAY_TO_STRING(dupes,','), 1, 50), '...')
      ELSE ARRAY_TO_STRING(dupes,',') END
    ) AS sample
    FROM duplicate_data
    LIMIT 5;

![](http://ww2.sinaimg.cn/large/a490147fgw1f5rawd6ei2j20in06n0uy.jpg)

### Cities with the same name in the same country and region

Worldwide, even in the same region of a country, there can be multiple cities with the exact same name. Take for example Georgetown, in Indiana. Geonames says there are 3 towns with that name in Indiana. Wikipedia says there are even more:

*   [Georgetown, Floyd County, Indiana](https://en.wikipedia.org/wiki/Georgetown,_Floyd_County,_Indiana)
*   [Georgetown Township, Floyd County, Indiana](https://en.wikipedia.org/wiki/Georgetown_Township,_Floyd_County,_Indiana)
*   [Georgetown, Cass County, Indiana](https://en.wikipedia.org/wiki/Georgetown,_Cass_County,_Indiana)
*   [Georgetown, Randolph County, Indiana](https://en.wikipedia.org/wiki/Georgetown,_Randolph_County,_Indiana)

    WITH duplicate_data AS (
        SELECT
        city_name,
        array_agg(ROW(country_code, region_code)) AS dupes
        FROM city_region_data
        WHERE country_code = 'US'
        GROUP BY city_name, region_code, country_code
        ORDER BY COUNT(ROW(country_code, region_code)) DESC
    )
    SELECT
    city_name,
    ARRAY_LENGTH(dupes, 1) AS duplicity,
    ( CASE WHEN ARRAY_LENGTH(dupes,1) > 9 
      THEN CONCAT(SUBSTRING(ARRAY_TO_STRING(dupes,','), 1, 50), '...')
      ELSE ARRAY_TO_STRING(dupes,',') END
    ) AS sample
    FROM duplicate_data
    LIMIT 4;

![](http://ww2.sinaimg.cn/large/a490147fgw1f5raxacpo0j20d505rmy4.jpg)

## Reverse geocoding

Both `(city_name, country_code)` and `(city_name, country_code, region_name)` tuples have failed as candidates to uniquely identify a location. We would have the option of using [zip codes](https://en.wikipedia.org/wiki/ZIP_code) or [postal codes](https://en.wikipedia.org/wiki/Postal_code) except we can't use those since most geolocation services don't offer that. But most geolocation services do offer longitude and latitude, and we can use those to eliminate ambiguity.

### Geometric data types in PostgreSQL

I looked further into the PostgreSQL docs and found that it also has geometric [data types](https://www.postgresql.org/docs/9.4/static/datatype-geometric.html) and [functions](https://www.postgresql.org/docs/9.4/static/functions-geometry.html)for 2D geometry. Out of the box you can model points, boxes, paths, polygons, circles, you can store them and query them. PostgreSQL has [some additional extensions](https://www.postgresql.org/docs/9.1/static/contrib.html) in the contrib directory. They are available out of the box with most Postgres installs. In this situation we're interested in the [cube](https://www.postgresql.org/docs/9.4/static/cube.html) and [earthdistance](https://www.postgresql.org/docs/9.4/static/earthdistance.html) extensions <sup>[2](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fn.2)</sup>. The cube extension allows you to model [n-dimensional](https://en.wikipedia.org/wiki/Real_coordinate_space#Vector_space) vectors, and the earthdistance extension uses [3-cubes](https://en.wikipedia.org/wiki/Hypercube) to store vectors and represent points on the surface of the Earth. We'll be using the following:

*   the `earth_distance` function is available, and it allows you to compute the [great-circle distance](https://en.wikipedia.org/wiki/Great-circle_distance)between two points
*   the `earth_box` function to check if a point is within a certain distance of a reference point
*   a [gist](https://www.postgresql.org/docs/9.1/static/sql-createindex.html) [expression index](https://www.postgresql.org/docs/9.4/static/indexes-expressional.html) on the expression `ll_to_earth(lat, long)` to make fast [spatial queries](https://en.wikipedia.org/wiki/Spatial_query) and find nearby points

### Designing a view for city & region data

Geonames data was imported into 3 tables:

*   `geo_geoname` (data from [cities1000.zip](http://download.geonames.org/export/dump/cities1000.zip) )
*   `geo_admin1` (data from [admin1CodesASCII.txt](http://download.geonames.org/export/dump/admin1CodesASCII.txt) )
*   `geo_countryinfo` (data from [countryInfo.txt](http://download.geonames.org/export/dump/countryInfo.txt) )

Then we create a view that pulls everything together <sup>[3](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fn.3)</sup>. We now have population data, city/region/country data, and lat/long data, all in one place.

    CREATE OR REPLACE VIEW city_region_data AS ( 
        SELECT
            b.country AS country_code,
            b.asciiname AS city_name,
            a.name AS region_name,
            b.region_code,
            b.population,
            b.latitude AS city_lat,
            b.longitude AS city_long,
            c.name    AS country_name
        FROM geo_admin1 a
        JOIN (
            SELECT *, (country || '.' || admin1) AS country_region, admin1 AS region_code
            FROM geo_geoname
            WHERE fclass = 'P'
        ) b ON a.code = b.country_region
        JOIN geo_countryinfo c ON b.country = c.iso_alpha2
    );

### Designing a nearby-city query and function

In the most nested `SELECT`, we're only keeping the cities in a 23km radius around the reference point, then we're applying a country filter and city pattern filter (these two filters are optional), and we're only getting the closest 50 results to the reference point. Next, we're reordering by population because geonames sometimes has districts and neighbourhoods around bigger cities <sup>[4](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fn.4)</sup>, and it does not mark them in a specific way, so we just want to select the larger city and not a district (for example let's say the geolocation service returned a lat/long that would resolve to one distrct of a larger metropolitan area. In my case, I'd like to resolve this to the larger city it's associated with) We're also creating a gist index (the `@>` operator will make use of the gist index) which we're using to find points within a radius of a reference point. This function takes a point (using latitude and longitude) and returns the city, region and country that is associated with that point.

    CREATE INDEX geo_geoname_latlong_idx ON geo_geoname USING gist(ll_to_earth(latitude,longitude));
    CREATE OR REPLACE FUNCTION geo_find_nearest_city_and_region(
        latitude double precision,
        longitude double precision,
        filter_countries_arr varchar[],
        filter_city_pattern  varchar,
    ) RETURNS TABLE(
        country_code varchar,
        city_name varchar,
        region_name varchar,
        region_code varchar,
        population bigint,
        _lat double precision,
        _long double precision,
        country_name varchar,
        distance numeric
        ) AS $
    BEGIN
        RETURN QUERY
        SELECT *
        FROM (
            SELECT
            *
            FROM (
                SELECT 
                *,
                ROUND(earth_distance(
                       ll_to_earth(c.city_lat, c.city_long),
                       ll_to_earth(latitude, longitude)
                      )::numeric, 3) AS distance_
                FROM city_region_data c
                WHERE earth_box(ll_to_earth(latitude, longitude), 23000) @> ll_to_earth(c.city_lat, c.city_long) AND
                      (filter_countries_arr IS NULL OR c.country_code=ANY(filter_countries_arr)) AND
                      (filter_city_pattern  IS NULL OR c.city_name LIKE filter_city_pattern)
                ORDER BY distance_ ASC
                LIMIT 50
            ) d
            ORDER BY population DESC
        ) e
        LIMIT 1;
    END;
    $
    LANGUAGE plpgsql;

## Conclusion

We've started from the design of a system that would query multiple geoip services, would gather the data and would then [aggregate](https://en.wikipedia.org/wiki/Aggregate_data) it to get a more reliable result. We first looked at some ways of uniquely identifying locations. We've then picked a way that would eliminate ambiguity in identifying them. In the second half, we've looked at different ways of structuring, storing and querying geographical data in PostgreSQL. Then we've built a view and a function to find cities near a reference point which allowed us to do reverse geocoding.

## Footnotes:

<sup>[1](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fnr.1)</sup>

By using multiple services (and assuming they use different data sources internally) after aggregation, we can have a more reliable answer than if we were using just one.

Another advantage here is that we're using free services, no setup is required, we don't have to take care of updates, since these services are maintained by their owners.

However, querying all these web services will be slower than querying a local geoip data structures. But, there are city/country/region geolocation database out there such as [geoip2](https://www.maxmind.com/en/geoip2-databases) from maxmind,[ip2location](http://www.ip2location.com/databases/db3-ip-country-region-city) or [db-ip](https://db-ip.com/db/#downloads).

<sup>[2](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fnr.2)</sup>

There's a nice [post here](http://tapoueh.org/blog/2013/08/05-earthdistance) using the `earthdistance` module to compute distances to nearby or far away pubs.

<sup>[3](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fnr.3)</sup>

Geonames has geonameIds as well, which are geonames-specific ids we can use to accurately refer to locations.

<sup>[4](https://blog.garage-coding.com/2016/07/06/geolocation-using-multiple-services.html#fnr.4)</sup>

geonames does not have polygonal data about cities/neighbourhoods or metadata about the type of urban area (like openstreetmap does) so you can't query all city polygons (not districts/neighbourhoods) that contain that point.

