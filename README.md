# nashville-housing
DATA CLEANING USING SQL: The following steps were taken to make this data cleaner and easier to analyze and use.  Each of these steps demonstrates the SQL Code necessary to perform the operations that are specified.

Step 1: Cleaning the Date Field.  

The Sale Date Field began with a format that looks like this:   [2013-04-09 00:00:00.000]
 
Step 1 is to remove all of the zeroes because we do not need to know the time that the sale took place and all of the time fields are zeroes anyway.  

	ALTER TABLE NashvilleHousing
	ALTER COLUMN [SaleDate] date

Date now appears as 2013-04-09

Step 2: Populate Property Addresses that are null

There are 35 rows where the Property Address is null. On a list of housing sales, that is a problem. Upon further examination, there are duplicate Parcel ID numbers and in one row the address is blank, but in the other row, it is not.  We will develop an update statement that will populate the null rows with the addresses from the other rows, only in the situation where both rows have the same Parcel ID #.

We use the following language to query and then update the table: 
	SELECT *
	FROM PortfolioProject.dbo.NashvilleHousing
	ORDER BY ParcelID


SELECT a.ParcelID, a.PropertyAddress, b.ParcelID, b.PropertyAddress, ISNULL(a.PropertyAddress,b.PropertyAddress)
FROM PortfolioProject.dbo.NashvilleHousing a
JOIN PortfolioProject.dbo.NashvilleHousing b
	ON a.ParcelID = b.ParcelID
	AND a.[UniqueID ] <> b.[UniqueID ]
WHERE a.PropertyAddress is null


UPDATE a
SET PropertyAddress = ISNULL(a.PropertyAddress,b.PropertyAddress)
FROM PortfolioProject.dbo.NashvilleHousing a
JOIN PortfolioProject.dbo.NashvilleHousing b
	ON a.ParcelID = b.ParcelID
	AND a.[UniqueID ] <> b.[UniqueID ]
WHERE a.PropertyAddress is null

Now there are no longer any null property addresses in our table.


Step 3: Breaking Out Property Address into Individual Columns using SUBSTRING

The “PropertyAddress” column contains both the street address and the city name in one column. It would be better to have them separated into two columns.  We can do this by using the following queries and ALTER commands.

	SELECT PropertyAddress
	FROM PortfolioProject.dbo.NashvilleHousing

	SELECT 
		SUBSTRING (PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1) AS Address
		, SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) AS City
	FROM PortfolioProject.dbo.NashvilleHousing

	ALTER TABLE NashvilleHousing
	ADD PropertySplitAddress Nvarchar(255), PropertySplitCity Nvarchar(255);

	UPDATE NashvilleHousing
	SET PropertySplitAddress = SUBSTRING (PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1)
	, PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress));
	
	SELECT *
	FROM PortfolioProject.dbo.NashvilleHousing

	ALTER TABLE NashvilleHousing
	DROP COLUMN PropertyAddress;


STEP 4: Breaking Out Owner Address Column using PARSENAME

Our table also contains columns for the Owner Address, which may be different from the Property Address. The Owner Address Column contains Street Addresses, Cities and States, so it must be broken down into 3 new columns. The following queries and ALTER commands will accomplish this using the PARSENAME command.

	SELECT OwnerAddress
	FROM  PortfolioProject.dbo.NashvilleHousing

	SELECT
		PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3) AS OwnerAddr
		,PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2) AS OwnerCity
		,PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1) AS OwnerState
	FROM PortfolioProject.dbo.NashvilleHousing

	ALTER TABLE NashvilleHousing
	ADD OwnerAddr Nvarchar(255), OwnerCity Nvarchar(255), OwnerState Nvarchar(255);

	UPDATE NashvilleHousing
	SET OwnerAddr = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3),
		OwnerCity = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2),
		OwnerState = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1);


	SELECT *
	FROM PortfolioProject.dbo.NashvilleHousing

	ALTER TABLE NashvilleHousing
	DROP COLUMN OwnerName, OwnerAddress;

 	-- The OwnerName was also deleted to protect individuals’ privacy.


Step 5: Change Y and N to Yes and No in “Sold As Vacant Field”

There are inconsistencies in this field, because some values were entered as “Y” or “N” and other times they were entered as “Yes” or “No”.

We updated the table using the following CASE Statement

	SELECT Distinct(SoldAsVacant), Count(SoldasVacant)
	FROM  PortfolioProject.dbo.NashvilleHousing
	GROUP BY SoldAsVacant
	ORDER BY 2

	SELECT SoldAsVacant,
	CASE When SoldAsVacant = 'Y' THEN 'Yes'
	 	When SoldAsVacant = 'N' THEN 'No'
	 	ELSE SoldAsVacant
	 	END
	FROM  PortfolioProject.dbo.NashvilleHousing

	UPDATE NashvilleHousing
	SET SoldAsVacant = CASE When SoldAsVacant = 'Y' THEN 'Yes'
	 WHEN SoldAsVacant = 'N' THEN 'No'
	 ELSE SoldAsVacant
	 END


Step 6: Remove Duplicates

After visually examining the data closely and looking for duplicates, it is obvious that some records contain identical information in multiple fields.  These records need to be removed in order to get accurate analytics.

This SQL code was  used to query for duplicates and then remove them from the table. It used a Common Table Expression

	-- First Identify Duplicates
	
	WITH RowNumCTE AS(
		SELECT *,
			ROW_NUMBER() OVER (
			PARTITION BY	ParcelID,
							PropertySplitAddress,
							SalePrice,
							SaleDate,
							LegalReference
		ORDER BY 
			UniqueID
			) row_num
									
	FROM  PortfolioProject.dbo.NashvilleHousing
	)
	SELECT *
	FROM RowNumCTE
	WHERE row_num > 1
	ORDER BY PropertySplitAddress


-- Then Delete Them

	WITH RowNumCTE AS(
	SELECT *,
		ROW_NUMBER() OVER (
		PARTITION BY	ParcelID,
						PropertySplitAddress,
						SalePrice,
						SaleDate,
						LegalReference
		ORDER BY 
			UniqueID
			) row_num
									
	FROM  PortfolioProject.dbo.NashvilleHousing
	)

	DELETE 
	FROM RowNumCTE
	WHERE row_num > 1

Step 7: Delete Unused Columns

The last task is the simplest one.  Certain columns are unnecessary and will not be needed for analysis, so we can remove them using the following code:

	SELECT *
	FROM PortfolioProject.dbo.NashvilleHousing

	ALTER TABLE PortfolioProject.dbo.NashvilleHousing
	DROP COLUMN LegalReference, TaxDistrict, Bedrooms,FullBath, HalfBath


