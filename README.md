CREATE procedure Seller_Action_Pending    
as    
begin    
Declare @set_email2 nvarchar(max)    
declare @set_email nvarchar(max)    
    
 set @set_email = ',scsit.db3@sparecare.in,hanish.khattar@sparecare.in,manish.sharma@sparecare.in';          
 set @set_email2 = 'scsit.db3@sparecare.in,hanish.khattar@sparecare.in,manish.sharma@sparecare.in';     
    
    
    
WITH EmailAgg AS (              
    SELECT               
        LocationID,              
        STRING_AGG(CASE WHEN Sequence IN (1, 2) THEN Email END, ',') AS ToEmail,              
        STRING_AGG(CASE WHEN Sequence IN (3, 4, 5, 6) THEN Email END, ',') AS CcEmail              
    FROM LocationContactMatrixMapping              
    WHERE Sequence IN (1, 2, 3, 4, 5, 6)         
   GROUP BY LocationID              
)     
insert into INCORRECT_INVOICE(Seller,Buyer,DispatchOrderNo,LRNumber,LRDate,Value,Reason,ToEmail,ccemail)    
SELECT Concat(C.Dealer,'_',C.Location) Seller, Concat(D.Dealer,'_',D.Location) Buyer,    
A.DispatchOrderNo, A.LRNumber, convert(varchar,A.LRDate,105) as LRDate, A.SaleRealisationAmount as Value, A.RemarkForSA as Reason    
,eg.ToEmail,    
  case when eg.CcEmail is null then @set_email2 else  concat( Eg.CcEmail  ,@set_email) end as ccemail     
    
                              FROM SH_DispatchDetail A (NOLOCK)                                         
    
                              OUTER APPLY ( SELECT TOP 1 * FROM SH_PARTTRANSACTION (NOLOCK)        where DispatchOrderNo=A.DispatchOrderNo)B      
    
                              LEFT JOIN LocationInfo C (NOLOCK)             ON(B.SELLERLOCATION=C.LocationID)    
    
                              LEFT JOIN LocationInfo D (NOLOCK)             ON(B.BUYERLOCATION=D.LocationID)    
    
                              INNER JOIN SH_TRNCODEMASTER E (NOLOCK)   ON(A.TrnCode=E.TrnCode)    
    
                              LEFT JOIN LocationContactMatrixMapping F on C.locationID =F.LocationID    
    
         left join EmailAgg as eg on eg.LocationID = c.LocationID    
    
                              WHERE A.isSaleRealisation='N' and A.TrnCode is not NULL and A.StatusForSA ='Request'                                                                                   
    
                              AND CONVERT(DATE,GETDATE())>=CONVERT(DATE,DATEADD(DAY,C.SaleRealBufferDays+(SELECT COUNT(*) FROM HOLIDAY     
    
                              WHERE HDate BETWEEN A.DeliverDate AND CONVERT(DATE,GETDATE())),A.DeliverDate))    
    
                              and ManifestCheckStatus is NOT null and F.ConsigneePerson = 'y' and    
          f.WalletFb = 'Y'              
        AND (              
            LOWER(f.Designation) LIKE '%spm%' OR               
            LOWER(f.Designation) = 'spare parts manager'              
        )              
        AND f.Email IS NOT NULL               
        AND (f.Email <> ''               
        OR f.Email LIKE '%@%' )    
  and concat(a.dispatchorderno,a.lrnumber) not in    
  (select CONCAT(dispatchorderno,lrnumber) from INCORRECT_INVOICE)    
    
  end    
