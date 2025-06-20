# 📦 SDTM Demographics (DM) Dataset Creation

This document outlines the full process for creating a **CDISC SDTM-compliant Demographics (DM)** dataset from raw clinical trial data using SAS. The final dataset is saved both in SAS dataset format and as a transport `.XPT` file for submission.

---

## 📁 Input Files

1. `sdtm_demographics.xlsx`  
   Raw demographic data
2. `SAS Project SDTM -EX Raw data.xlsx`  
   Exposure data
3. `SAS Project SDTM-Rnd raw data.xlsx`  
   Randomization data

---

## 🔧 SAS Code Workflow

### 1. Import Demographics Data

```sas
PROC IMPORT DATAFILE="/home/Datasets/sdtm_demographics.xlsx"
    OUT=WORK.DM DBMS=XLSX REPLACE;
RUN;

```

### 2. Create DM1: Format and derive key SDTM variables

```sas
DATA DM1;
SET WORK.DM;
STUDYID=STRIP(STUDY);
DOMIN="DM";
SUBJID=STRIP(SUBJID);
SITEID=STRIP(SITE);
USUBJID=STRIP(STUDYID)||"-"||STRIP(SITEID)||"-"||STRIP(SUBJID);
RFSTDTC=PUT(ENTDT,IS8601DA.)||"T"||PUT(ENRTM,TOD8.);
RFENDTC=PUT(CMPDT,IS8601DA.)||"T"||PUT(CMPTM,TOD8.);
RFICDTC=PUT(infdt,is8601da.)||"T"||PUT(inftm,tod8.);
RFPENDTC=PUT(CMPDT,is8601da.)||"T"||PUT(CMPTM,tod8.);
DTHDTC="";
DTHFL="";
INVNAM=STRIP(inv);
RACE=upcase(etho);
AGE=AGE;
RUN;

```

### 3. Add AGEU and format SEX values

```sas
DATA DM2;
SET DM1;
IF AGE NE . THEN AGEU="YEARS";
RUN;

PROC SQL;
CREATE TABLE DM3 AS
SELECT *,
CASE
WHEN UPCASE(GEN) IN ("FEMALE") THEN 'F'
WHEN UPCASE(GEN) IN ("MALE") THEN 'M'
ELSE ''
END AS SEX FROM DM2;
RUN;

PROC PRINT DATA=DM3;
RUN;

```

### 4. Import Exposure Data and Derive Treatment Start/End

```sas
PROC IMPORT DATAFILE="/home/Datasets/SAS Project SDTM -EX Raw data.xlsx"
OUT=WORK.EX
DBMS=XLSX REPLACE;
RUN;

DATA EX1;
SET WORK.EX;
STUDYID=STRIP (STUDY);
DOMIN="DM";
SUBJID=STRIP (SUBJID);
SITEID=STRIP (SITE);
USUBJID=STRIP (STUDYID)||"-"||STRIP (SITEID)||"-"||STRIP (SUBJID);

DSDTN=INPUT (DSDT,MMDDYY10.);
RFXSTDTC=PUT (DSDTN,is8601da.)||"T"||PUT (DSDTM,tod8.);

IF VISIT="Period-1";
TRT1=TRT;
KEEP USUBJID RFXSTDTC TRT1;
RUN;

DATA EX2;
SET WORK.EX;
STUDYID=STRIP (STUDY);
DOMIN="DM";
SUBJID=STRIP (SUBJID);
SITEID=STRIP (SITE);
USUBJID=STRIP (STUDYID)||"-"||STRIP (SITEID)||"-"||STRIP (SUBJID);

DSDTN=INPUT (DSDT,MMDDYY10.);
RFXENDTC=PUT (DSDTN,is8601da.)||"T"||PUT (DSDTM,tod8.);

IF VISIT="Period-2";
TRT2=TRT;
KEEP USUBJID RFXENDTC TRT2;
RUN;

Merge EX1 and EX2 by USUBJID to get RFXSTDTC and RFXENDTC.

PROC SORT DATA=EX1;BY USUBJID;RUN;

PROC SORT DATA=EX2;BY USUBJID;RUN;

DATA EX;
MERGE EX1(IN=A) EX2(IN=B);
BY USUBJID;
IF A OR B;
IF RFXENDTC EQ "" THEN RFXENDTC=RFXSTDTC;
RUN;

```

### 5. Merge Exposure with Demographics
```sas
PROC SORT DATA=DM3; BY USUBJID; RUN;

DATA DM4;
MERGE DM3(IN=A) EX(IN=B);
BY USUBJID;
RUN;

```
### 6. Merge Randomization Data
```sas

PROC IMPORT DATAFILE="/home/Datasets/SAS Project SDTM-Rnd raw data.xlsx"
    OUT=WORK.RND DBMS=XLSX REPLACE;
RUN;

PROC SORT DATA=DM4; BY SUBJID; RUN;
PROC SORT DATA=WORK.RND; BY SUBJID; RUN;

DATA DM5;
MERGE DM4(IN=A) RND(IN=B DROP=ARMDA ARMA);
BY SUBJID;
IF A;
RUN;


```
### 7. Final Derivations

``` sas

DATA DM6;
SET DM5;
ARMCD = ARMDP;
ARM = ARMP;
ACTARMCD = "ATRT";
ACTARM = "ATRTA";
COUNTRY = "IND";
IF TRT1="REF" AND TRT2="TEST" THEN DO; ATRT="R-T"; ATRTA="REFE-TEST";END;
IF TRT1="TEST" AND TRT2="REF" THEN DO; ATRT="T-R"; ATRTA="TEST-REFE";END;
KEEP
STUDYID
DOMIN
USUBJID
SUBJID
SITEID
RFSTDTC
RFENDTC
RFXSTDTC
RFXENDTC
RFICDTC
RFPENDTC
DTHDTC
DTHFL
INVNAM
AGE
AGEU
SEX
RACE
ETHO
ARMCD
ARM
ACTARMCD
ACTARM
COUNTRY
;
RUN;

```

### 8. Create Final SDTM-DM Dataset with Labels

```sas

PROC SQL;
CREATE TABLE DM_FINAL AS
SELECT
STUDYID	"Study identifier"	LENGTH=	8 ,
DOMIN	"Domin Abbreviation"	LENGTH=	2 ,
USUBJID	"Unique Subject Identifier"	LENGTH=	50 ,
SUBJID	    "Subject Identifier for the Study"	LENGTH=	50 ,
SITEID	     "Study Site Identifier"	LENGTH=	20 ,
RFSTDTC	"Subject Reference Start Date/Time"	LENGTH=	25 ,
RFENDTC	"Subject Reference End Date/Time"	LENGTH=	25 ,
RFXSTDTC	"Date/Time of First study Treatment"	LENGTH=	25 ,
RFXENDTC	"Date/Time of Last study Treatment"	LENGTH=	25 ,
RFICDTC	 "Date/Time of Informed Consent" LENGTH= 25 ,
RFPENDTC	"Date/Time of End of Participation"	LENGTH=	25 ,
DTHDTC	"Date/Time of Death"	LENGTH=	25 ,
DTHFL	"Subject Death Flag" LENGTH= 2 ,
INVNAM	"Investigator Name" LENGTH=	100 ,
AGE	 "Age"	LENGTH=	8 ,
AGEU	"Age units"	LENGTH=	6 ,
SEX	 "Sex"	LENGTH=	2 ,
RACE	"Race"	LENGTH=	100 ,	
ARMCD	"Planned Arm Code"	LENGTH=	100 ,
ARM	"Description of Planned Arm"	LENGTH=	200 ,
ACTARMCD	"Actual Arm Code"	LENGTH=	100 ,
ACTARM	"Description of Actual Arm"	LENGTH=	200 ,
COUNTRY "Country" LENGTH=50
FROM DM6;
QUIT;

LIBNAME SDTM "/home/Datasets";
DATA SDTM.DM(LABEL="DEMOGRAPHY");
SET DM_FINAL;
RUN;


LIBNAME XPT XPORT "/home/Datasets/SDTM.DM.XPT";
DATA XPT.DM;
SET DM_FINAL;
RUN;


```
