# projecttest
Demo : https://thriving-torte-197d45.netlify.app/


@Repository
public class DossierKpiStatusViewRepositoryCustomImpl implements DossierKpiStatusViewRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<DossierStatusProjection> findAverageTimeByStage() {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();

        // Inner query: calculate MAX, MIN, elapsed time, and stage designation
        CriteriaQuery<Tuple> innerQuery = cb.createTupleQuery();
        Root<DossierStatusKpi> root = innerQuery.from(DossierStatusKpi.class);

        // Calculate MAX(exit_date) with fallback to CURRENT_DATE
        Expression<Date> maxExitDate = cb.coalesce(root.get("exitDate"), cb.currentDate());
        Expression<Long> maxExitDateEpoch = cb.function("EXTRACT", Long.class, cb.literal("EPOCH"), maxExitDate);

        // Calculate MIN(entry_date)
        Expression<Long> minEntryDateEpoch = cb.function("EXTRACT", Long.class, cb.literal("EPOCH"), root.get("entryDate"));

        // Calculate elapsed time: MAX(exit_date) - MIN(entry_date)
        Expression<Long> elapsedTime = cb.diff(maxExitDateEpoch, minEntryDateEpoch);

        // Case for 'designation'
        Expression<String> designation = cb.selectCase()
            .when(cb.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
            .otherwise(root.get("stage"));

        // Group by criteria
        innerQuery.multiselect(
                root.get("stageCode").alias("stageCode"),
                designation.alias("designation"),
                elapsedTime.alias("elapsedTime")
        ).groupBy(root.get("stageCode"), designation);

        // Outer query: calculate average elapsed time
        CriteriaQuery<Tuple> outerQuery = cb.createTupleQuery();
        Subquery<Tuple> subquery = outerQuery.subquery(Tuple.class);
        Root<Tuple> subRoot = subquery.correlate(innerQuery);

        // Calculate average elapsed time
        Expression<Double> avgElapsedTime = cb.avg(subRoot.get("elapsedTime"));
        Expression<Double> roundedAvgTime = cb.function("ROUND", Double.class, avgElapsedTime, cb.literal(0));

        // Final selection
        outerQuery.multiselect(
                subRoot.get("stageCode").alias("code"),
                subRoot.get("designation").alias("designation"),
                roundedAvgTime.alias("time")
        ).groupBy(subRoot.get("stageCode"), subRoot.get("designation"))
         .orderBy(cb.asc(subRoot.get("stageCode")));

        // Execute query and return results
        List<Tuple> result = entityManager.createQuery(outerQuery).getResultList();

        return result.stream()
            .map(tuple -> new DossierStatusProjectionImpl(
                tuple.get("code", String.class),
                tuple.get("designation", String.class),
                tuple.get("time", Double.class)
            ))
            .toList();
    }
}







................
@Repository
public class DossierKpiStatusViewRepositoryCustomImpl implements DossierKpiStatusViewRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<DossierStatusProjection> findAverageTimeByStage() {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Tuple> query = cb.createTupleQuery();
        Root<DossierStatusKpi> root = query.from(DossierStatusKpi.class);

        // Define case statement for 'designation'
        Expression<String> designation = cb.selectCase()
            .when(cb.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
            .otherwise(root.get("stage"));

        // Aggregate functions for MIN(entry_date) and MAX(exit_date)
        Expression<java.util.Date> maxExitDate = cb.greatest(root.get("exitDate"));
        Expression<java.util.Date> minEntryDate = cb.least(root.get("entryDate"));

        // Calculate elapsed time using MAX(exit_date) - MIN(entry_date)
        Expression<Long> elapsedTime = cb.diff(
            cb.function("UNIX_TIMESTAMP", Long.class, cb.coalesce(maxExitDate, cb.currentTimestamp())),
            cb.function("UNIX_TIMESTAMP", Long.class, minEntryDate)
        );

        // Average elapsed time
        Expression<Double> averageElapsedTime = cb.avg(elapsedTime);

        // Include MIN(entry_date) and MAX(exit_date) directly in query
        query.multiselect(
                root.get("stageCode").alias("code"),
                designation.alias("designation"),
                averageElapsedTime.alias("averageTime"),
                minEntryDate.alias("minEntryDate"),
                maxExitDate.alias("maxExitDate")
            )
            .groupBy(root.get("stageCode"), designation)
            .orderBy(cb.asc(root.get("stageCode")));

        // Execute query and map results to DTO or Projection
        List<Tuple> result = entityManager.createQuery(query).getResultList();

        return result.stream()
            .map(tuple -> new DossierStatusProjectionImpl(
                tuple.get("code", String.class),
                tuple.get("designation", String.class),
                tuple.get("averageTime", Double.class),
                tuple.get("minEntryDate", java.util.Date.class),
                tuple.get("maxExitDate", java.util.Date.class)
            ))
            .toList();
    }
}






...............


public class DossierKpiStatusSpecifications {

    public static Specification<DossierKpiStatus> calculateElapsedTime() {
        return (root, query, criteriaBuilder) -> {
            // Subquery for MAX exit_date (or CURRENT_TIMESTAMP if null)
            Subquery<LocalDateTime> maxExitDate = query.subquery(LocalDateTime.class);
            Root<DossierKpiStatus> maxRoot = maxExitDate.from(DossierKpiStatus.class);
            maxExitDate.select(
                criteriaBuilder.coalesce(maxRoot.get("exitDate"), criteriaBuilder.currentTimestamp())
            ).where(criteriaBuilder.equal(maxRoot.get("uuid"), root.get("uuid")));

            // Subquery for MIN entry_date
            Subquery<LocalDateTime> minEntryDate = query.subquery(LocalDateTime.class);
            Root<DossierKpiStatus> minRoot = minEntryDate.from(DossierKpiStatus.class);
            minEntryDate.select(
                minRoot.get("entryDate")
            ).where(criteriaBuilder.equal(minRoot.get("uuid"), root.get("uuid")));

            // Main query: calculate elapsed time and average it
            query.multiselect(
                root.get("stageCode").alias("stageCode"),
                criteriaBuilder.selectCase()
                    .when(criteriaBuilder.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
                    .otherwise(root.get("stageDesignation")).alias("designation"),
                criteriaBuilder.avg(
                    criteriaBuilder.function(
                        "TIMESTAMPDIFF", Long.class, // Database-dependent; adjust for portability
                        criteriaBuilder.literal("SECOND"),
                        minEntryDate.getSelection(),
                        maxExitDate.getSelection()
                    )
                ).alias("averageElapsedTime")
            );

            // Group by stageCode and designation
            query.groupBy(
                root.get("stageCode"),
                criteriaBuilder.selectCase()
                    .when(criteriaBuilder.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
                    .otherwise(root.get("stageDesignation"))
            );

            query.orderBy(criteriaBuilder.asc(root.get("stageCode")));

            return null; // Specifications require null here to build a query
        };
    }
}







public static Specification<Task> calculateAverageElapsedTimeWithoutDirectJoin() {
    return (Root<Task> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
        // Create a subquery or join with a manual condition for RefStatus
        Root<RefStatus> refStatusRoot = query.from(RefStatus.class);

        // Join condition: task.dossierCodeStatus == refStatus.code
        Predicate joinCondition = cb.equal(root.get("dossierCodeStatus"), refStatusRoot.get("code"));

        // Join RefStage from RefStatus
        Join<RefStatus, RefStage> refStageJoin = refStatusRoot.join("refStage", JoinType.INNER);

        // CASE logic for designation
        Expression<String> designation = cb.selectCase()
            .when(cb.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
            .otherwise(refStageJoin.get("designation"));

        // Time calculation: COALESCE(exitDate, CURRENT_DATE) - entryDate
        Expression<Long> elapsedTime = cb.diff(
            cb.function("coalesce", Date.class, root.get("exitDate"), cb.literal(Date.from(LocalDate.now().atStartOfDay(ZoneId.systemDefault()).toInstant()))),
            root.get("entryDate")
        );

        Expression<Double> elapsedTimeInHours = cb.quot(
            cb.function("extract_epoch", Long.class, elapsedTime),
            3600
        );

        // Grouping fields
        query.groupBy(designation, refStageJoin.get("code"), refStageJoin.get("id"));

        // Select fields
        query.multiselect(
            designation.alias("designation"),
            refStageJoin.get("code").alias("code"),
            cb.avg(elapsedTimeInHours).alias("averageElapsedTime")
        );

        // Add join condition to the WHERE clause
        query.where(joinCondition);

        // Ordering
        query.orderBy(cb.asc(refStageJoin.get("id")));

        return null; // Predicate not needed for this query
    };
}





***********

import org.springframework.data.jpa.domain.Specification;

import javax.persistence.criteria.*;
import java.time.LocalDate;

public class TaskViewSpecifications {

    public static Specification<TaskView> calculateAverageElapsedTimeWithMinMax(Long specificStatusId) {
        return (Root<TaskView> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {

            // COALESCE(exitdate, CURRENT_DATE)
            Expression<LocalDate> exitDate = cb.coalesce(root.get("exitDate"), cb.literal(LocalDate.now()));

            // MIN(entryDate) and MAX(exitDate)
            Expression<LocalDate> minEntryDate = cb.least(root.get("entryDate"));
            Expression<LocalDate> maxExitDate = cb.greatest(exitDate);

            // CASE to handle specific status and create 'Fake Stage'
            Expression<String> stageId = cb.selectCase()
                    .when(cb.equal(root.get("status"), specificStatusId), cb.literal("Fake Stage"))
                    .otherwise(root.get("stageId"));

            // Calculate elapsed time: (maxExitDate - minEntryDate) in hours
            Expression<Long> elapsedTimeInSeconds = cb.function(
                    "EXTRACT", Long.class,
                    cb.literal("EPOCH"),
                    cb.diff(maxExitDate, minEntryDate)
            );
            Expression<Double> elapsedTimeInHours = cb.quot(elapsedTimeInSeconds, 3600.0);

            // Group by stage_id
            query.groupBy(stageId);

            // Select stage_id and average elapsed time
            query.multiselect(
                    stageId, // Stage ID or Fake Stage
                    cb.avg(elapsedTimeInHours) // Average elapsed time in hours
            );

            query.orderBy(cb.asc(stageId)); // Order results by stage ID

            return null; // No specific filtering condition
        };
    }
}
