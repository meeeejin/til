# Summary of LinkBench for MySQL

## MySQL schema

- 4 indexes created across 3 tables

## Tables

```bash
$ ls -al | grep ibd
-rw-r----- 1 mijin mijin  3074424832 10월 16 11:19 counttable.ibd
-rw-r----- 1 mijin mijin  3418357760 10월 16 11:18 linktable#P#p0.ibd
-rw-r----- 1 mijin mijin  2512388096 10월 16 11:19 linktable#P#p10.ibd
-rw-r----- 1 mijin mijin  1191182336 10월 16 11:19 linktable#P#p11.ibd
-rw-r----- 1 mijin mijin  2751463424 10월 16 11:19 linktable#P#p12.ibd
-rw-r----- 1 mijin mijin  5981077504 10월 16 11:19 linktable#P#p13.ibd
-rw-r----- 1 mijin mijin  2365587456 10월 16 11:19 linktable#P#p14.ibd
-rw-r----- 1 mijin mijin  2264924160 10월 16 11:19 linktable#P#p15.ibd
-rw-r----- 1 mijin mijin  1279262720 10월 16 11:18 linktable#P#p1.ibd
-rw-r----- 1 mijin mijin  2420113408 10월 16 11:18 linktable#P#p2.ibd
-rw-r----- 1 mijin mijin  3539992576 10월 16 11:18 linktable#P#p3.ibd
-rw-r----- 1 mijin mijin  2193620992 10월 16 11:18 linktable#P#p4.ibd
-rw-r----- 1 mijin mijin  7327449088 10월 16 11:19 linktable#P#p5.ibd
-rw-r----- 1 mijin mijin  3133145088 10월 16 11:19 linktable#P#p6.ibd
-rw-r----- 1 mijin mijin  2139095040 10월 16 11:19 linktable#P#p7.ibd
-rw-r----- 1 mijin mijin  2503999488 10월 16 11:19 linktable#P#p8.ibd
-rw-r----- 1 mijin mijin  1342177280 10월 16 11:19 linktable#P#p9.ibd
-rw-r----- 1 mijin mijin 18824036352 10월 16 11:20 nodetable.ibd
```

### linktable

```sql
CREATE TABLE `linktable` (
  `id1` bigint(20) unsigned NOT NULL DEFAULT '0',
  `id2` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `visibility` tinyint(3) NOT NULL DEFAULT '0',
  `data` varchar(255) NOT NULL DEFAULT '',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` int(11) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (link_type, `id1`,`id2`),
  KEY `id1_type` (`id1`,`link_type`,`visibility`,`time`,`id2`,`version`,`data`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY key(id1) PARTITIONS 16;
```

- A primary key on `(link_type, id1, id2)`
- A secondary key on `(id1, link_type, visibility, time, id2, version, data)`
- The linktable is a generic association or bridge table allowing any two nodes to be associated in some way
- The secondary index covers for the most frequent query
    - It almost doubles the size of linktable
    - But, it also reduces the worst-case random I/O required for that query from a few thousand to a few
- `link_type`: A specific relationship between any two nodes (e.g., users being friends, a user liking a post, a user that is tagged in a photo, etc.)

### counttable

```sql
CREATE TABLE `counttable` (
  `id` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `count` int(10) unsigned NOT NULL DEFAULT '0',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` bigint(20) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`,`link_type`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

- A primary key on `(id, link_type)`
- **Very important for performance and scalability in a social network**
- The counttable maintains counts of a given link type for a node
- Counts are transactionally updated and this small investment in the form of *additional writes* pays off by allowing for *quick access* to the number of likes, shares, posts, friends and other associations between nodes
    - Without it, the application would have to continuously query the DB to retrieve up-to-date count info creating a large amount of system load

### nodetable

```sql
CREATE TABLE `nodetable` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `type` int(10) unsigned NOT NULL,
  `version` bigint(20) unsigned NOT NULL,
  `time` int(10) unsigned NOT NULL,
  `data` mediumtext NOT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

- A primary key on `(id)`
- The nodetable defines an object or end-point within the social graph
- Nodes include users, comments, replies, photos, etc.
- `type`: What the node represents
- `data`: The object itself (up to 16M)

## Reference

- [Summary of Linkbench for MongoDB & MySQL](http://smalldatum.blogspot.com/2015/07/summary-of-linkbench-for-mongodb-mysql.html)
- [MongoDB / TokuMX plugin for LinkBench (Part 1)](https://rimzy.net/category/linkbench)
- [Facebook LinkBench Tests PostgreSQL Social Relation Profile Scenario Performance](https://alibaba-cloud.medium.com/facebook-linkbench-tests-postgresql-social-relation-profile-scenario-performance-42d05abc50a7)