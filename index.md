# Using UKDF Qserv instance as FrDF Rubin Science Platform TAP service backend

```{abstract}
In this note, we describe the procedure to set up a Qserv instance hosted remotely as the backend for the Rubin Science Platform (RSP) TAP service hosted at CC-IN2P3. We also present some tests conducted to evaluate the impact on performance linked to this configuration.
```

## Introduction
The Rubin Science Platform (RSP) is a set of web applications providing easy access to Rubin data for interactive analysis. Access to the data is mainly provided by an IVOA Table Access Protocol [^1], which provides an interface between the user and the backend catalog database.

The catalog database, in general, is stored on a Qserv instance [^2].
The FrDF RSP currently uses its own Qserv instance, which is deployed on a Kubernetes cluster hosted on local bare-metal machines, and provides access to 4 catalogs with a total of 55TB of data.

We are exploring the possibility to use a Qserv instance deployed outside the CC-IN2P3 as backend for the FrDF RSP TAP service.

## RSP TAP configuration for  Qserv access

In this section we descrive how to configure TAP in Phalanx to use a Qserv instance. 

### Phalanx TAP configuration

The TAP RSP service can be easily configured to point to a Qserv instance via:

```
 qserv:
      host: "hostname:port"
```

But this configuration is not enough to make the TAP service able to talk with Qserv. TAP also needs to know the Qserv databases' schemas, and this information is provided via a Docker image built by the Qserv admins (who know what Qserv serves) and provided to the TAP service via the following configuration:

```
cadc-tap:
  tapSchema:
    image:
      repository: "image"
      tag: x.y.z
```

In thiw way, we the right settings, we can then setup an RSP TAP service to talk with any Qserv instance.

### TAP Configuration at FrDF

The  FrDF  RSP is configured to use the FrDF Qserv instance in TAP. 

The configuraton is described in the next block of code: 

```
cadc-tap:
  tapSchema:
    image:
      repository: "gabrimaine/tap-schema-ccin2p3"
      tag: 2.4.1


config:
    qserv:
      host: "ccqserv201.in2p3.fr:30040"
```

The configuration point to the FrDF Qserv instance and use as TAP schema image the image tap-schema-ccin2p3 providing the DB schema for all the catalogs ingested in the FrDF Qserv. 

For our test, we wanted check the configuration and the performance of the TAP service using a remote Qserv instance. For this, we set up the TAP service to point to the UKDF Qserv. This implied a change in the Phalanx configuration but also a change in the UKDF firewall to provide access to the UKDF Qserv from FrDF RSP IPs.

Once the problem with the firewall was fixed, the FrDF RSP TAP configuration was changed to:

```
cadc-tap:
  tapSchema:
    image:
      repository: "stvoutsin/tap-schema-roe"
config:
	 qserv:
      host: "192.41.122.85:30040"
```


## Performance test

The main objective was to test the impact on query performance using a remote Qserv instance instead of the local Qserv instance.
This has been done by performing a set of queries on FrDF RSP using both Qserv instances (FrDF and UKDF) as the backend

The queries used are listed in the following table.  

```{rst-class} technote-wide-content
```

 | Query Index | Query |
 | ----- | ----- |
 | 1     | ``` SELECT diasrc.ra, diasrc.decl, diasrc.diaObjectId, diasrc.diaSourceId, diasrc.filterName, diasrc.midPointTai,scisql_nanojanskyToAbMag(diasrc.psFlux) AS psAbMag,ccdvis.seeing,ccdvis.visitId FROM dp02_dc2_catalogs.DiaSource AS diasrc``` |
 |       |``` JOIN dp02_dc2_catalogs.CcdVisit AS ccdvis ON diasrc.ccdVisitId = ccdvis.ccdVisitIdWHERE CONTAINS(POINT('ICRS', diasrc.ra, diasrc.decl), CIRCLE('ICRS', 67.4579, -44.0802, 0.0010))=1AND diasrc.filterName = 'i'```|
 | 2     |```SELECT objectId, coord_ra, coord_dec, detect_isPrimary, scisql_nanojanskyToAbMag(g_cModelFlux) as gmag, scisql_nanojanskyToAbMag(i_cModelFlux) as imag,scisql_nanojanskyToAbMag(r_cModelFlux) as rmag,scisql_nanojanskyToAbMagSigma(g_cModelFlux, g_cModelFluxErr) as gmag_err,``` |   
 |       | ```scisql_nanojanskyToAbMagSigma(i_cModelFlux, i_cModelFluxErr) as imag_err,scisql_nanojanskyToAbMagSigma(r_cModelFlux, r_cModelFluxErr) as rmag_err```|
 |       |```FROM dp02_dc2_catalogs.ObjectWHERE CONTAINS (POINT('ICRS', coord_ra, coord_dec), CIRCLE('ICRS', 62.0, -37.0, 0.4)) = 1AND detect_isPrimary = 1```|
 | 3     | ```SELECT * FROM dp02_dc2_catalogs.Object WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec),CIRCLE('ICRS', 63.01804167, -32.87422222, 0.027777777777777776))=1```|
 | 4     | ```SELECT TOP 200000 coord_ra, coord_dec```|
 |       | ```FROM dp02_dc2_catalogs.Object WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec), CIRCLE('ICRS', 62, -37, 1)) = 1 AND detect_isPrimary = 1```|
 | 5     | ```SELECT RADIANS(coord_ra) AS ra_radians, RADIANS(coord_dec) AS dec_radians```|
 |       | ``` FROM dp02_dc2_catalogs.Object WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec), CIRCLE('ICRS', 62, -37, 1)) = 1 AND detect_isPrimary =1```|
 | 6     | ```SELECT scisql_nanojanskyToAbMag(g_cModelFlux) AS gmag, scisql_nanojanskyToAbMag(r_cModelFlux) AS rmag, scisql_nanojanskyToAbMag(i_cModelFlux) AS imag```|
 |       | ```FROM dp02_dc2_catalogs.Object WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec), CIRCLE('ICRS', 62, -37, 1)) = 1 AND detect_isPrimary != 0 AND scisql_nanojanskyToAbMag(i_cModelFlux) <= 25```|
 | 7     | ```SELECT scisql_nanojanskyToAbMag(g_cModelFlux) AS gmag, scisql_nanojanskyToAbMag(r_cModelFlux) AS rmag, scisql_nanojanskyToAbMag(i_cModelFlux) AS imag```|
 |       | ```FROM dp02_dc2_catalogs.Object WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec), CIRCLE('ICRS', 62, -37, 1)) = 1 AND detect_isPrimary = 1 AND r_cModelFlux BETWEEN g_cModelFlux AND i_cModelFlux```|
 | 8     | ```SELECT objectId, scisql_nanojanskyToAbMag(g_psfFlux) AS gmag, scisql_nanojanskyToAbMag(r_psfFlux) AS rmag, scisql_nanojanskyToAbMag(r_psfFlux) AS imag, scisql_nanojanskyToAbMag(i_psfFlux) AS zmag```|
 |       |```FROM dp02_dc2_catalogs.Object ```|
 |       |```WHERE objectId IN (1249537790362809267, 1252528461990360512, 1248772530269893180, 1251728017525343554, 1251710425339299404, 1250030371572068167, 1253443255664678173, 1251807182362538413, 1252607626827575504, 1249784080967440401, 1253065023664713612, 1325835101237446771)```|
 | 9     | ```SELECT scisql_nanojanskyToAbMag(fs.psfFlux) AS mag, scisql_nanojanskyToAbMagSigma(fs.psfFlux, fs.psfFluxErr) AS magerr, cv.band, cv.expMidptMJD```|
 |       | ```FROM dp02_dc2_catalogs.Object AS o JOIN dp02_dc2_catalogs.ForcedSource AS fs ON o.objectId = fs.objectId JOIN dp02_dc2_catalogs.CcdVisit AS cv ON fs.ccdVisitId = cv.ccdVisitId```|
 |       |```WHERE CONTAINS(POINT('ICRS', o.coord_ra, o.coord_dec), CIRCLE('ICRS', 62.1479031, -35.799138, 0.0006)) = 1 AND o.detect_isPrimary = 1```|
 | 10    | ```SELECT mt.id_truth_type AS mt_id_truth_type,mt.match_objectId AS mt_match_objectId,ts.ra AS ts_ra,ts.dec AS ts_dec,ts.truth_type AS ts_truth_type,ts.mag_r AS ts_mag_r,```|
 |       | ```ts.is_pointsource AS ts_is_pointsource,ts.redshift AS ts_redshift,ts.flux_u AS ts_flux_u,ts.flux_g AS ts_flux_g,ts.flux_r AS ts_flux_r,ts.flux_i AS ts_flux_i,ts.flux_z AS ts_flux_z,```|
 |       | ```ts.flux_y AS ts_flux_y,obj.coord_ra AS obj_coord_ra,obj.coord_dec AS obj_coord_dec,obj.refExtendedness AS obj_refExtendedness,scisql_nanojanskyToAbMag(obj.r_cModelFlux) AS obj_cModelMag_r,```|
 |       |```obj.u_cModelFlux AS obj_u_cModelFlux,obj.g_cModelFlux AS obj_g_cModelFlux,obj.r_cModelFlux AS obj_r_cModelFlux,obj.i_cModelFlux AS obj_i_cModelFlux,obj.z_cModelFlux AS obj_z_cModelFlux,obj.y_cModelFlux AS obj_y_cModelFlux```|
 |       | ```FROM dp02_dc2_catalogs.MatchesTruth AS mt JOIN dp02_dc2_catalogs.TruthSummary AS ts ON mt.id_truth_type=ts.id_truth_type JOIN dp02_dc2_catalogs.Object AS obj ON mt.match_objectId=obj.objectId```|
 |       |```WHERE scisql_s2PtInCircle(obj.coord_ra,obj.coord_dec,62.0,-37.0,0.1)=1 AND ts.truth_type=1 AND obj.detect_isPrimary=1```|
 | 11    | ```SELECT id_truth_type, ra, dec, host_galaxy FROM dp02_dc2_catalogs.TruthSummary WHERE id_truth_type = 'MS_9684_23_3'```| 
 | 12    | ```SELECT coord_ra, coord_dec FROM dp02_dc2_catalogs.Object``` |
 |       |```WHERE CONTAINS(POINT('ICRS', coord_ra, coord_dec), POLYGON('ICRS',48.57,-44.63,75.24,-44.63,75.24,-26.78,48.57,-26.78 )) = 1 AND r_extendedness = 1 AND detect_isPrimary = 1 AND scisql_nanojanskyToAbMag(r_cModelFlux) < 21.0```|
 | 13    | ```SELECT fs.forcedSourceId,fs.objectId,fs.ccdVisitId,fs.detect_isPrimary,fs.band,scisql_nanojanskyToAbMag(fs.psfFlux) AS psfMag,ccd.obsStartMJD,scisql_nanojanskyToAbMag(obj.r_psfFlux) AS obj_rpsfMag```|
 |       |```FROM dp02_dc2_catalogs.ForcedSource AS fs JOIN dp02_dc2_catalogs.CcdVisit AS ccd ON fs.ccdVisitId=ccd.ccdVisitId JOIN dp02_dc2_catalogs.Object AS obj ON fs.objectId=obj.objectId```|
 |       |```WHERE scisql_s2PtInCircle(obj.coord_ra,obj.coord_dec,62,-37,1.0)=1 AND obj.detect_isPrimary=1 AND obj.r_extendedness=0 AND scisql_nanojanskyToAbMag(obj.r_cModelFlux) > 17.5 AND scisql_nanojanskyToAbMag(obj.r_cModelFlux) < 18.0 AND fs.band='r'```|
 | 14    | # Catalog Queries with the TAP Service notebook|


The query number 14 listed above is not a single query, but it consists of running [a full notebook](https://github.com/rubin-dp0/tutorial-notebooks/blob/main/DP02_02b_Catalog_Queries_with_TAP.ipynb) provided for the DP0.2 test in RSP Nublado.

The queries against TAP service have been performed in different ways

1. Using the RSP Portal ADQL interface
2. Using an external service (Topcat) via token provided by the RSP
3. Using an RSP Nublado notebook for query 14 as described above. 

We also tested the same queries on USDF and IDF RSP (only via ADQL Portal interface). 


### Results 

The following table reports the results (in seconds) for the tested requests. Not all the queries have been tested with Topcat and they are marked as NP. 
The table's columns are the following: 

1. **Query Index** is the identifier of the query as listed in the table above.
2. **# Sources** is the number of sources retrieved by the query 
3. **Qserv Chunks** is the number of Qserv chunks scanned by the query
4. **Qserv Time** is the time needed by Qserv to execute the query as reported by the Qserv dashboard
5. **FrDF/UKDF Portal** is the time measured by the Portal to perform the query 
6. **FrDF/UKDF Topcat** is the time needed to perform the query via Topcat
7. **IDF/USDF** is the time to perform the query via Portal on IDF/USDF instances 


```{rst-class} technote-wide-content
```

 | Query Index | # Sources | Qserv Chunks | Qserv Time | FrDF via Portal | FrDF via TOPCAT | UKDF via Portal | UKDF via TOPCAT | IDF | USDF |
 | :---------: |:--------: | :-----------:| :---------:| :-------------: | :-------------: | :-------------: | :-------------: |:---:|:----:|
 | 1           | 15        | 1            | 1          | 2               | 2               | 3               | 3               | 2   | 1    |
 | 2           | 50000     | 6            | 2          | 8               | 14              | 9               | 7               | 6   | 5    |
 | 3           | 1519      | 1            | 2          | 17              | 10              | 18              | 11              | 17  | 8    |
 | 4           | 200000    | 19           | 6          | 11              | NP              | 10              | NP              | 11  | 5    |
 | 5           | 200000    | 19           | 5          | 13              | NP              | 11              | NP              | 9   | 8    |
 | 6           | 200000    | 19           | 5          | 11              | NP              | 12              | NP              | 12  | 7    |
 | 7           | 500000    | 19           | 8          | 23              | 22              | 24              | 25              | 18  | 12   |
 | 8           | 12        | 11           | 1          | 5               | 3               | 3               | 4               | 3   | 1    |
 | 9           | 432       | 1            | 6          | 9               | NP              | 9               | NP              | 3   | 2    |
 | 10          | 14501     | 2            | 136        | 143             | 143             | 159             | 149             | 147 | 84   |
 | 11          | 1         | 1            | 1          | 2               | NP              | 2               | NP              | 2   | 1    |
 | 12          | 500000    | 1393         | 27         | 71              | NP              | 79              | NP              | 482 | 13   |
 | 13          | 41751     | 19           | 1          | 5               | 3               | 5               | 5               | 16  | 6    |
 | 14          | NA        | NA           | NA         |80               | NA              | 78              | NA              | 210 | 48   |


### Queries via Portal 
The next plot shows the results for queries executed using the ADQL interface directly from the RSP Portal service. The queries have been performed via three RSP instances: IDF, USDF, and FrDF. FrDF instances have been configured with FrDF Qserv and UKDF Qserv as backend.

```{figure} ./images/tap_time.png

ADQL Queries via TAP Portal interface for IDF, USDF, and FrDF instances. FrDF RSP configured with FrDF and UKDF Qserv backend.
```

We note a generally better performance of USDF RSP, but for the other RSP instances, there are no significant differences (excluding query number 12, removed in the next plot, which seems to have some problem with IDF).

```{figure} ./images/tap_time_no_outliers.png

ADQL Queries via TAP Portal interface for IDF, USDF, and FrDF instances. FrDF RSP configured with FrDF and UKDF Qserv backend. Query #12 removed to improve readability.
```

For the scope of our exercise, the results do not show a loss in performance when moving from FrDF Qserv to UKDF Qserv, nor do they indicate a significant impact from network latency on queries. The time for each query is consistent between FrDF and UKDF Qserv.

These results seem to confirm that using UKDF Qserv as the FrDF RSP backend is a viable option. 
However, there are still some uncertainties about the load on the UKDF Qserv: specifically, how the impact of UKDF users can affect the RSP performance at FrDF and vice versa. This point is currently not measurable and requires further study.


### Queries via Topcat

Results for the two FrDF configurations are confirmed also using TAP service for Topcat [^3] queries.

```{figure} ./images/topcat_vs_portal.png

ADQL Queries via Topcat using TAP service provided by FrDF RSP configured with FrDF and UKDF Qserv backend.
```
The time needed to process a query is almost the same whether the RSP backend is set to point to the FrDF Qserv instance or the UKDF Qserv instance.


### Queries processing 

We noted, however, an impact not due to the Qserv backend but probably due to the 'post-processing' of results by the Portal (and Topcat), affecting the time necessary to display them. Comparing the query time as measured at the level of Qserv and the time measured at the level of the Portal, we sometimes see a noticeable discrepancy, as shown in the next figure.

```{figure} ./images/qserv_vs_portal.png

Time to process a query at the level of Qserv (as reported by the Qserv dashboard) and at the level of the Portal.
```

Topcat is also affected by this 'post-processing' issue but seems a little less impacted than Portal.

## Conclusions

The scope of the test was checking the impact on performance for our RSP (FrDF RSP) when a remote Qserv instance (UKDF) is used as the backend for the TAP service. 
The first preliminary test seems to exclude a large impact on performance due to, e.g., network latency, load on Qserv, etc. However, we cannot exclude an impact on this aspect when the load on the UKDF Qserv increases. 
The performance is in line using Portal and Topcat TAP services, but we noted a delay in results showing, probably due to 'post-processing' steps needed by Portal and Topcat to organize the retrieved catalogs.

Further tests are needed to verify these results with a set of more articulated and 'science representative' queries, but for the moment we can confirm the possibility of using a remote Qserv instance as an RSP backend.


[^1]: https://dmtn-264.lsst.io/
[^2]: https://dmtn-243.lsst.io/
[^3]: https://www.star.bris.ac.uk/~mbt/topcat/
