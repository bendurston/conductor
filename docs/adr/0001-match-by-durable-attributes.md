# Match items by durable content attributes, not service IDs

Conductor matches a Source item to its Destination equivalent using durable content attributes —
ISRC for tracks, UPC and/or normalized name+artist for albums, normalized name for artists and
playlists — rather than service-specific IDs. Service IDs would be simpler to handle but do not
survive crossing from one Service to another, which is the entire problem Conductor solves.
Matching is fully automated: a candidate at or above the Confidence threshold is accepted, and
anything below is reported as a distinct "no confident match" outcome rather than guessed or
silently dropped. There is no interactive disambiguation in v1.
