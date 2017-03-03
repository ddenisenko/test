# Algorithm of Merging Traits and Resource Types

This chapter describes algorithm of Merging Traits and Resource Types in pseudo-code.

The algorithm has three phases: 1. YAML-level merging 2. Parameters application 3. Reference patching

## Phases 1&2: YAML-level merging and Parameters application

It is more convenient to perform these two phases simultaneously, as if phase 2 introduces any new resource type or trait references, phase 1 needs to be repeated. It is possible to perform the phases one by one.

Please note that the algorithm is intentionally simplified.

```js

function expand(yamlRoot : YAMLNode) {

  //root is always YAMLMap, and root can have no counterparts from traits.
  expand(<YAMLMap>yamlRoot, [])
}

class TransformationInfo {
    templateName : string
    templateValue: string
    valueSource : RAMLUnit
}

class StringTransformer {
    transformations : TransformationInfo[]

    /**
      * Transforms arbitrary string. Returns transformation result.
      * If any replacements were made, reports the transformation to an optional report acceptor,
      * where start and end are the resulting positions from the beginning from the transformed string,
      * and valueSource is the unit, where the replacement originates from.
      *
      * Besides using the set of transformations also accepts built-in parameters like resourcePathName.
      * valueSource is null in case of built-in parameter.
      *
      * Also applies built-in transformation functions like !singularize
      */
    transform(input : string,
      transformInfoAcceptor? : (start: int, end : int, valueSource:RAMLUnit)=>void
    ) : string {
      //...
    }
}

class Counterpart {
    node : YAMLMap
    transformers : StringTransformer[]
}

/**
 * Expands node.
 * nodes - node counterparts from traits and resource types in the order of application.
 */
function expandNode(target: YAMLMap, counterparts : Counterpart[]){

    //MAP children to merge. From target unique ID to target MAP node
    var recursiveMergeTargets : { [id:string]:YAMLMap} = {}
    //MAP children to merge. From target unique ID to list of MAP nodes to merge
    var recursiveMergeNodes : { [id:string]:Counterpart[]} = {}

    //going through all node counterparts from traits and resource types one by one
    counterparts.forEach(currentCounterpart=>{

        //checking all mappings of the counterparts
        currentCounterpart.node.mappings.forEach(counterpartMapping=>{
            //looking for a mapping in the original node with the same key
            var originalMapping = findMappingByKey(target.mappings, counterpartMapping.key)

            if (originalMapping) {
                if (originalMapping.value.kind == Kind.Scalar) {

                    //mappings with scalar types from original node remain unchanged
                } else if (originalMapping.value.kind == Kind.Map) {

                    //we have nodes to be merged: counterpart should be merged to target
                    var targetNodeToMerge = originalMapping.value;
                    var counterpartNodeToMerge = counterpartMapping.value;

                    //saving target node
                    recursiveMergeTargets[targetNodeToMerge.uniqueId()] = targetNodeToMerge;

                    //saving counterpart node by adding to the list
                    var counterPartNodes = recursiveMergeNodes[targetNodeToMerge.uniqueId()];
                    if (!counterPartNodes) {
                        counterPartNodes = [];
                        recursiveMergeNodes[targetNodeToMerge.uniqueId()] = counterPartNodes;
                    }

                    counterPartNodes.push(counterpartNodeToMerge);
                } else if (originalMapping.value.kind == Kind.Sequence) {
                    //TODO
                }
            } else {

                //if no mapping with the same key found, simply adding current mapping
                //to the target
                var counterpartMappingCopy = counterpartMapping.copy();
                target.mappings.push(counterpartMappingCopy)
            }
        })
    })

    //continue to mappings with MAP type, we need to merge
    for(var targetNodeId in recursiveMergeTargets) {
        //target child MAP node to merge
        var childTarget = recursiveMergeTargets[targetNodeId];

        //its counterpart MAP nodes from traits and resource types to merge
        var childCounterparts = recursiveMergeNodes[targetNodeId];

        //if node is a method, resource, trait, or resource type, it can have
        // "is" and "type" statements pointing to other traits and resource types, recursively
        if (isMethod(childTarget) || isTrait(childTarget)
          || isResource(childTarget) || isResourceType(childTarget)) {

            //calling findCounterparts, which searches for any traits and resource type, current node depends on,
            //recursively, and returning their root MAP nodes.
            var additionalCounterparts = findCounterparts(childTarget)
        }

        //recursive merging
        expandNode(childTarget, childCounterparts)
    }
}



```
