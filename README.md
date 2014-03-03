meshBalancing
=============

This contains a single library, dynamicRefineBalancedFvMesh which is 
based on dynamicRefineFvMesh but adds mesh balancing for parallel cases
to the update() function.

Note: The redistributeParPlus function does NOT work. For some reason it does not
redistribute the cellLevel and pointLevel fields (hexRef8.distribute crashes) so
it is only redistributing the refinement history. The cellLevel volScalarField is
mapped, but pointLevel is a bit more tricky. The new version of redistributePar
in OpenFOAM-2.3.x appears to have fixed this, but I have not yet tested it.

A workaround may be to reconstruct the cellLevel and pointLevel manually from the
distributed refinement history.


## Usage

To use the balancing library with an existing 'DyM' solver you have to make the following
edits to the case (in addition to the changes to the source code in the section below)

  1. Add a linking statement to the new library in controlDict
  
        libs
        (
            "libdynamicFvMesh-dev.so"
        );

  2. Add the balanceParDict dictionary to the system folder (example in the library folder)
  3. Change the mesh type in dynamicMeshDict to `dynamicRefineBalancedFvMesh`
  4. Add the following two entries to the dynamicRefineFvMeshCoeffs dictionary (see the example
     in the library folder)
  
        enableBalancing true;
        allowableImbalance 0.15;

## Notes

To use this with interDyMFoam, you have to move the call to createPrghCorrTypes
inside correctPhi.H to avoid a crash when the number of patches changes.

## OpenFOAM Source Changes (for 2.3.x)

  1. [ __CRASH__ ] Add guard in src/dynamicMesh/polyTopoChange/refinementHistory.C in
     refinementHistory::distribute at line 927 to catch
     when fully un-refined cells are transferred
     
        if( newVisibleCells[i] >= 0 )
        {
            visibleCells_[constructMap[i]] = newVisibleCells[i] + offset;
        }
        
  2. [ __ANNOYANCE__ ] Add `if (debug)` guard around print statements in
     `refinementHistory::countProc` in the same file as edit #1 to prevent excessive
     printing to stdout during mesh balancing operations
     
  3. [ __CRASH__ ] Add the following to src/finiteVolume/fvMesh/nearWallDist.C
     at `nearWallDist::correct()` inside the
     `mesh_.changing()` condition but before the patch sizes are set. If the
     total number of patches on a processor increases, there is a crash unless
     we increase the size of nearWallDist first.
          
        // If the number of patches on this processor increased, we need to
        // increase the number of patches in nearWallDist to match
        if( mesh_.boundary().size() != size() )
        {
            //Resize nearWallDist to match mesh patches
            resize(mesh_.boundary().size());

            //Set any newly created (unset) patches
            forAll(*this, patchI)
            {
                if( !set(patchI) )
                {
                    set
                    (
                        patchI, 
                        fvPatchField<scalar>::New
                        (
                            calculatedFvPatchScalarField::typeName,
                            mesh_.boundary()[patchI],
                            mesh_.V()
                        )
                    );
                }
            }
        }
        
  4. [ __WARNINGS__ ] There was a small change in version 2.2.x in
     src/dynamicMesh/fvMeshAdder/fvMeshAdderTemplates.C which results in an
     excessive number of warnings. In the `MapVolField` function, there are
     three calls to `calcPatchMap`. The map given to the patches may contain
     unmapped cells until the last mapping. To supress the warnings, change the
     unmapped value entry in the first two calls from -1 to 0 (the value it
     used to be).

     
  5.  [ __CRASH__ ] DimensionedFields are not properly distributed in the current implementation. To enable distribution of DimensionedFields you will have to make the following changes:
      
    1. Add a constructor for DimensionedField which is consistent with the one
       used in fvMeshSubset. I recommend you recompile the entire src directory 
       after this step.
         
       **src/OpenFOAM/fields/DimensionedFields/DimensionedField/DimensionedField.H**

       _line 151ish, add:_

            //- Construct from dictionary
            DimensionedField
            (
                const IOobject&,
                const Mesh& mesh,
                const dictionary& fieldDict,
                const word& fieldDictEntry="value"
            );
            
       **src/OpenFOAM/fields/DimensionedFields/DimensionedField/DimensionedFieldIO.C**

       _line 82ish, add:_

            template<class Type, class GeoMesh>
            Foam::DimensionedField<Type, GeoMesh>::DimensionedField
            (
                const IOobject& io,
                const Mesh& mesh,
                const dictionary& fieldDict,
                const word& fieldDictEntry
            )
            :
                regIOobject(io),
                Field<Type>(0),
                mesh_(mesh),
                dimensions_(dimless)
            {
                readField(fieldDict, fieldDictEntry);
            }
      
    2. Add an interpolate template to fvMeshSubset that can handle
       a DimensionedField<type,volMesh> input
      
        **src/finiteVolume/fvMesh/fvMeshSubset/fvMeshSubset.H**

        _somewhere in the public functions, add:_

            //- Map dimensioned fields
            template<class Type>
            static tmp<DimensionedField<Type, volMesh> >
            interpolate
            (
                const DimensionedField<Type, volMesh>&,
                const fvMesh& sMesh,
                const labelList& cellMap
            );
            
            template<class Type>
            tmp<DimensionedField<Type, volMesh> >
            interpolate
            (
                const DimensionedField<Type, volMesh>&
            ) const;
            
        **src/finiteVolume/fvMesh/fvMeshSubset/fvMeshSubsetInterpolate.C**

        _line 38ish, add:_

            template<class Type>
            tmp<DimensionedField<Type, volMesh> > fvMeshSubset::interpolate
            (
                const DimensionedField<Type, volMesh>& df,
                const fvMesh& sMesh,
                const labelList& cellMap
            )
            {
                // Create and map the internal-field values
                Field<Type> internalField(df, cellMap);

                // Create the complete field from the pieces
                tmp<DimensionedField<Type, volMesh> > tresF
                (
                    new DimensionedField<Type, volMesh>
                    (
                        IOobject
                        (
                            "subset"+df.name(),
                            sMesh.time().timeName(),
                            sMesh,
                            IOobject::NO_READ,
                            IOobject::NO_WRITE
                        ),
                        sMesh,
                        df.dimensions(),
                        internalField
                    )
                );

                return tresF;
            }


            template<class Type>
            tmp<DimensionedField<Type, volMesh> > fvMeshSubset::interpolate
            (
                const DimensionedField<Type, volMesh>& df
            ) const
            {
                return interpolate
                (
                    df,
                    subMesh(),
                    cellMap()
                );
            }
      
    3. Update the field mapping in fvMeshAdder
      
        **src/dynamicMesh/fvMeshAdder/fvMeshAdder.H**

        _in the private functions, add:_

            //- Update single dimensioned Field.
            template<class Type>
            static void MapDimField
            (
                const mapAddedPolyMesh& meshMap,

                DimensionedField<Type, volMesh>& fld,
                const DimensionedField<Type, volMesh>& fldToAdd
            );

        _in the public functions, add:_

            //- Map all dimensionedFields of Type
            template<class Type>
            static void MapDimFields
            (
                const mapAddedPolyMesh&,
                const fvMesh& mesh,
                const fvMesh& meshToAdd
            );

        **src/dynamicMesh/fvMeshAdder/fvMeshAdderTemplates.C**

        _anywhere in the member functions section, add:_

            template<class Type>
            void Foam::fvMeshAdder::MapDimField
            (
                const mapAddedPolyMesh& meshMap,

                DimensionedField<Type, volMesh>& fld,
                const DimensionedField<Type, volMesh>& fldToAdd
            )
            {
                const fvMesh& mesh = fld.mesh();

                // Internal field
                // ~~~~~~~~~~~~~~

                {
                    // Store old internal field
                    Field<Type> oldInternalField(fld);

                    // Modify internal field
                    Field<Type>& intFld = fld;

                    intFld.setSize(mesh.nCells());

                    intFld.rmap(oldInternalField, meshMap.oldCellMap());
                    intFld.rmap(fldToAdd, meshMap.addedCellMap());
                }
            }


            template<class Type>
            void Foam::fvMeshAdder::MapDimFields
            (
                const mapAddedPolyMesh& meshMap,
                const fvMesh& mesh,
                const fvMesh& meshToAdd
            )
            {
                // This is a fix for the fact that the lookupClass function
                // now returns all the internal fields of GeometricFields when
                // called for a DimensionedField, but the typeName approach
                // differentiates between them.
                
                const wordList dimFieldNames
                (
                    mesh.names(DimensionedField<Type, volMesh>::typeName)
                );
                
                const wordList dimFieldNamesToAdd
                (
                    meshToAdd.names(DimensionedField<Type, volMesh>::typeName)
                );
                
                
                HashTable<const DimensionedField<Type, volMesh>&> fields
                (
                    dimFieldNames.size()
                );
                
                HashTable<const DimensionedField<Type, volMesh>&> fieldsToAdd
                (
                    dimFieldNamesToAdd.size()
                );
                
                forAll(dimFieldNames, i)
                {
                    fields.insert
                    (
                        dimFieldNames[i],
                        mesh.lookupObject<DimensionedField<Type, volMesh> >
                        (
                            dimFieldNames[i]
                        )
                    );
                }
                
                forAll(dimFieldNamesToAdd, i)
                {
                    fieldsToAdd.insert
                    (
                        dimFieldNamesToAdd[i],
                        meshToAdd.lookupObject<DimensionedField<Type, volMesh> >
                        (
                            dimFieldNamesToAdd[i]
                        )
                    );
                }

                for
                (
                    typename HashTable<const DimensionedField<Type, volMesh>&>::
                        iterator fieldIter = fields.begin();
                    fieldIter != fields.end();
                    ++fieldIter
                )
                {
                    DimensionedField<Type, volMesh>& fld =
                        const_cast<DimensionedField<Type, volMesh>&>
                        (
                            fieldIter()
                        );

                    if (fieldsToAdd.found(fld.name()))
                    {
                        const DimensionedField<Type, volMesh>& fldToAdd =
                            fieldsToAdd[fld.name()];

                        MapDimField<Type>(meshMap, fld, fldToAdd);
                    }
                    else
                    {
                        WarningIn("fvMeshAdder::MapDimFields(..)")
                            << "Not mapping field " << fld.name()
                            << " since not present on mesh to add"
                            << endl;
                    }
                }
            }

        **src/dynamicMesh/fvMeshAdder/fvMeshAdder.C**

        _line 104ish, in the `add` function, add:_

            fvMeshAdder::MapDimFields<scalar>(mapPtr, mesh0, mesh1);
            fvMeshAdder::MapDimFields<vector>(mapPtr, mesh0, mesh1);
            fvMeshAdder::MapDimFields<sphericalTensor>(mapPtr, mesh0, mesh1);
            fvMeshAdder::MapDimFields<symmTensor>(mapPtr, mesh0, mesh1);
            fvMeshAdder::MapDimFields<tensor>(mapPtr, mesh0, mesh1);
            
    4. Add in the actual distribution if the different types of
         DimensionedFields in fvMeshDistribute.
         
        **src/dynamicMesh/fvMeshDistribute/fvMeshDistribute.C**
                
        _line 1527ish, add:_
            
            const wordList dimScalarFields
            (
                mesh_.names(DimensionedField<scalar, volMesh>::typeName)
            );
            checkEqualWordList("dimScalarFields", dimScalarFields);
            
            const wordList dimVectorFields
            (
                mesh_.names(DimensionedField<vector, volMesh>::typeName)
            );
            checkEqualWordList("dimVectorFields", dimVectorFields);
            
            const wordList dimSphericalTensorFields
            (
                mesh_.names(DimensionedField<sphericalTensor, volMesh>::typeName)
            );
            checkEqualWordList("dimSphericalTensorFields", dimSphericalTensorFields);
            
            const wordList dimSymmTensorFields
            (
                mesh_.names(DimensionedField<symmTensor, volMesh>::typeName)
            );
            checkEqualWordList("dimSymmTensorFields", dimSymmTensorFields);
            
            const wordList dimTensorFields
            (
                mesh_.names(DimensionedField<tensor, volMesh>::typeName)
            );
            checkEqualWordList("dimTensorFields", dimTensorFields);
            
            
            
        _line 1790ish (immediately after the sendFields<surfaceTensorField>, add:_
            
            sendFields<DimensionedField<scalar,volMesh> >
            (
                recvProc,
                dimScalarFields,
                subsetter,
                str
            );
            sendFields<DimensionedField<vector,volMesh> >
            (
                recvProc,
                dimVectorFields,
                subsetter,
                str
            );
            sendFields<DimensionedField<sphericalTensor,volMesh> >
            (
                recvProc,
                dimSphericalTensorFields,
                subsetter,
                str
            );
            sendFields<DimensionedField<symmTensor,volMesh> >
            (
                recvProc,
                dimSymmTensorFields,
                subsetter,
                str
            );
            sendFields<DimensionedField<tensor,volMesh> >
            (
                recvProc,
                dimTensorFields,
                subsetter,
                str
            );
            
        _line 1970ish (after the PtrList<surfaceTensorField> line), add:_

            PtrList<DimensionedField<scalar,volMesh> > isf;
            PtrList<DimensionedField<vector,volMesh> > ivf;
            PtrList<DimensionedField<sphericalTensor,volMesh> > istf;
            PtrList<DimensionedField<symmTensor,volMesh> > isytf;
            PtrList<DimensionedField<tensor,volMesh> > itf;
            
        _line 2080ish (after the receiveFields<surfaceTensorField> call), add:_

            receiveFields<DimensionedField<scalar,volMesh> >
            (
                sendProc,
                dimScalarFields,
                domainMesh,
                isf,
                fieldDicts.subDict(DimensionedField<scalar,volMesh>::typeName)
            );
            receiveFields<DimensionedField<vector,volMesh> >
            (
                sendProc,
                dimVectorFields,
                domainMesh,
                ivf,
                fieldDicts.subDict(DimensionedField<vector,volMesh>::typeName)
            );
            receiveFields<DimensionedField<sphericalTensor,volMesh> >
            (
                sendProc,
                dimSphericalTensorFields,
                domainMesh,
                istf,
                fieldDicts.subDict(DimensionedField<sphericalTensor,volMesh>::typeName)
            );
            receiveFields<DimensionedField<symmTensor,volMesh> >
            (
                sendProc,
                dimSymmTensorFields,
                domainMesh,
                isytf,
                fieldDicts.subDict(DimensionedField<symmTensor,volMesh>::typeName)
            );
            receiveFields<DimensionedField<tensor,volMesh> >
            (
                sendProc,
                dimTensorFields,
                domainMesh,
                itf,
                fieldDicts.subDict(DimensionedField<tensor,volMesh>::typeName)
            );
            
            
  6.  [ __INFREQUENT CRASH__ ] The version of scotch shipped with OpenFOAM-2.1.x (Version 5.1.11)
      has a bug that will show up sometimes when you are balancing the mesh. The error will say
      something to the effect of `dgraphFoldComm: internal error (3)`. Sometimes restarting the run
      is enough to recover, but occassionally it cannot get past that. The solution is to upgrade
      scotch to 5.1.12 (do NOT upgrade to 6.0.0, it is not compatible with OpenFOAM-2.1.x).

      To do this upgrade, download and extract the source files for scotch 5.1.12 into 
      ThirdParty-2.1.x/scotch_5.1.12. Next, change OpenFOAM-2.1.x/etc/config/scotch.sh to point
      to this new folder. Clean the third party directory with its `Allclean` and recompile it
      with its `Allwmake`. Then go to OpenFOAM-2.1.x/src/parallel and clean it with `wclean` and
      `rmdepall`. Recompile it with its `Allwmake`. Then, just to be sure, go to OpenFOAM-2.1.x/src
      and run `Allwmake`.

  7.  [ __METHOD ERROR__ ] In the current implementation of dynamicRefineFvMesh, on which
      this is based, a refined cell can be coarsened if the minimum refinementField value
      in any of its child cells is less than the threshhold in the dictionary. This is
      nonsense and leads to oscillatory refinement. Consider the following cell, showing the
      value of `refinementField` in each cell
      
         +------+------+
         |      |      |
         |  0   | 9.9  |
         |      |      |
         +------+------+
         |      |      |
         | 9.9  | 9.9  |
         |      |      |
         +------+------+
         
     where refinement is triggered at a value of 10 and unrefinement at a value of, say, 0.5. 
     This cell, rather than being left
     alone, will be coarsened. It should use the maximum value rather than the minimum
     value. To change this, in `dynamicRefineFvMesh.H` on line 112 change `minCellField`
     to `maxCellField`. In `dynamicRefineFvMesh.C`, make the following 4 changes. Line 
     numbers here may not match your line numbers exactly, so use some common sense.
     Changes 1-3 will all be in the existing `minCellField` function and change 4 is the only
     location in the file that referenced the `minCellField` function.
     
     1. Line 646: change `minCellField` to `maxCellField`
     2. Line 648: change `GREAT` to `-GREAT`
     3. Line 656: change `min` to `max`
     4. Line 1236: change `minCellField` to `maxCellField`

  8. [ __CRASH__ ] When using viscosity models other than Newtonian in multiphase systems, each
     model creates a field named "nu" which conflict with each other when re-balancing the mesh.
     To fix it, for example in `BirdCarreau.C`, change `"nu"` on line 79 to `"BirdCarreau::"+name`
     so the field has a unique name. A similar modification can be made for the other viscosity
     models if needed.

